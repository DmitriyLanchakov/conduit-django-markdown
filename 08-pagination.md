# Pagination

One way to ensure users don’t come back is to make your app nice and slow. There is evidence to suggest that the typical user will abandon your app if it takes more than three seconds to load. With so many things to do, who has time to sit around and wait for 10 second API calls to complete?

Why is this important? Because pagination is one way we can lower response times and improve performance.

For those not familiar with pagination, the idea is simple: You probably have too much data to load all at once, so break that data up into multiple pages. We get the first 20 results and then the next 20 and so on.

Why does this increase performance? Because, when working with a large number of documents — say 10,000 or more — the database is not the bottleneck. Sure, if you’re suffering from an N+1 query problem, you’re going to be in trouble. But we’re smart people. For us, the bottleneck will be serializing the documents and downloading them in the browser. Serializing is done in Python, which makes the process inherently slow. The extent to which the browser downloading the response is a problem depends on the total size of the response. If 10,000 documents comes to a whopping 10kb, you’re probably alright. On the other hand, if it turns out to be 1MB of data, things are going to get nasty.

By breaking our data into smaller sections, we can avoid these problems altogether. Serializing is fast enough when working on small data sets and, if 20 documents adds up to 1MB for your use case, you’re probably reading the wrong book.

We alluded to retrieving 20 results at a time. This is an arbitrary number, but is considered a good starting point in most cases. You should feel free adjust this number as you see fit.

## Pagination Settings

Since we were just talking about page size, it makes sense to start by adding two new settings. Open up `conduit/settings.py` and scroll to the bottom. Make this change to the `REST_FRAMEWORK` settings:

```diff
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'conduit.apps.authentication.backends.JWTAuthentication',
    ),
+    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'EXCEPTION_HANDLER': 'conduit.apps.core.exceptions.core_exception_handler',
    'NON_FIELD_ERRORS_KEY': 'error',
+    'PAGE_SIZE': 20,
}
```

{x: pagination default_pagination_class setting}
Add the `DEFAULT_PAGINATION_CLASS` setting to `settings.REST_FRAMEWORK`.

{x: pagination page_size setting}
Add the `PAGE_SIZE` setting to `settings.REST_FRAMEWORK`.

The TL;DR (too long; didn’t read) here is that DRF offers multiple pagination strategies. The one we’ve chosen is the `LimitOffsetPagination` strategy. This will allow the client to pass in two query parameters to any endpoint that returns multiple documents: `limit` and `offset`. `limit` refers to the number of documents to be returned. `offset` is where we should start looking for documents to return for this page. The `PAGE_SIZE` setting tells us that the default value for `limit` will be `20`.

{x: pagination look into pagination strategies}
Check out the different [pagination strategies](#) we get from DRF.

As an example, consider a user is browsing a list of articles. The user has already been through four pages of articles and is now loading the fifth page. The client hasn’t specified the `limit` query parameter, so we’re using the default of `20`. Because the user was just looking at the fourth page of articles, we know that we’ve already shown the user 80 articles (4 pages * 20 documents per page). For the fifth page, we want to show documents 80-99, we the client will have to pass `offset` as `80`. That’s how the server knows to ignore results 0-79 when deciding what to respond with. If the client doesn’t pass `offset`, we will simply return results 0-19.

## Updating ConduitJSONRenderer

The next thing we need to do is update our renderer. We do support rendering lists of data right now, but it’s too hacky for my taste. Let’s clean it up.

Open up `conduit/apps/core/renderers.py` and make these changes:

```diff
import json

from rest_framework.renderers import JSONRenderer
-from rest_framework.utils.serializer_helpers import ReturnList


class ConduitJSONRenderer(JSONRenderer):
    charset = 'utf-8'
    object_label = 'object'
-    object_label_plural = 'objects'
+    pagination_object_label = ‘objects’
+    pagination_object_count = ‘count’

-    def render(self, data, media_type=None, renderer_context=None):
-        if isinstance(data, ReturnList):
-            _data = json.loads(
-                super(ConduitJSONRenderer, self).render(data).decode('utf-8')
-            )
-
-            return json.dumps({
-                self.object_label_plural: _data
-            })
-        else:
-            # If the view throws an error (such as the user can't be authenticated
-            # or something similar), `data` will contain an `errors` key. We want
-            # the default JSONRenderer to handle rendering errors, so we need to
-            # check for this case.
-            errors = data.get('errors', None)
-
-            if errors is not None:
-                # As mentioned about, we will let the default JSONRenderer handle
-                # rendering errors.
-                return super(ConduitJSONRenderer, self).render(data)
-
-            return json.dumps({
-                self.object_label: data
-            })

+       def render(self, data, media_type=None, renderer_context=None):
+        if data.get('results', None) is not None:
+            return json.dumps({
+                self.pagination_object_label: data['results'],
+                self.pagination_count_label: data['count']
+            })
+
+        # If the view throws an error (such as the user can't be authenticated
+        # or something similar), `data` will contain an `errors` key. We want
+        # the default JSONRenderer to handle rendering errors, so we need to
+        # check for this case.
+        elif data.get('errors', None) is not None:
+            return super(ConduitJSONRenderer, self).render(data)
+
+        else:
+            return json.dumps({
+                self.object_label: data
+            })
```

{x: pagination make renderer work with pagination}
Update `ConduitJSONRenderer` to work with our new paginated results.

These changes will let us handle rendering paginated data properly. The different here is that we’re checking for the `results` key instead of doing some arbitrary type checking for `ReturnList` that could change at any time and break out application.

## Updating the other renderers

Because we changed the properties expected by the renderer, we need to go through each renderer and update them to work again.

Let’s start with `conduit/apps/articles/renderers.py`. Make the following changes:

```diff
class ArticleJSONRenderer(ConduitJSONRenderer):
    object_label = 'article'
-    object_label_plural = 'articles'
+    pagination_object_label = ‘articles’
+    pagination_object_count = ‘articlesCount’


class CommentJSONRenderer(ConduitJSONRenderer):
    object_label = 'comment'
-    object_label_plural = 'comments'
+    pagination_object_label = ‘comments’
+    pagination_object_count = ‘commentsCount’
```

{x: pagination update articlejsonrenderer to work with conduitjsonrenderer}
Update `ArticleJSONRenderer` to work with the changes to `ConduitJSONRenderer`.

{x: pagination update commentjsonrenderer to work with conduitjsonrenderer}
Update `ConduitJSONRenderer` to work with the changes to `ConduitJSONRenderer`.

Now open `conduit/apps/authentication/renderers.py` and apply these changes:

```diff
class UserJSONRenderer(ConduitJSONRenderer):
    object_label = 'user'
+    pagination_object_label = 'users'
+    pagination_count_label = 'usersCount'

    def render(self, data, media_type=None, renderer_context=None):
        # …
```

{x: pagination update userjsonrenderer to work with the conduitjsonrenderer}
Update `UserJSONRenderer` to work with the changes to `ConduitJSONRenderer`.

Finally, open `conduit/apps/profiles/renderers.py` and make these changes:

```diff
class ProfileJSONRenderer(ConduitJSONRenderer):
    object_label = 'profile'
+    pagination_object_label = 'profiles'
+    pagination_count_label = 'profilesCount'
```

{x: pagination update profilejsonrenderer to work with conduitjsonrenderer}
Update `ProfileJSONRenderer` to work with the changes to `ConduitJSONRenderer`.

## Paginating Articles

A while back we wrote a custom `.list()` method for `ArticlesViewSet` because we needed to pass the `request` as context to the serializer. This is all well and good, but we broke pagination when we did that. DRF’s default `.list()` view supports pagination out of the box, but we’ll have to add it to our custom method.

Open `conduit/apps/articles/views.py`. Make these changes:

```diff
class ArticleViewSet(mixins.CreateModelMixin,
                     mixins.ListModelMixin,
                     mixins.RetrieveModelMixin,
                     viewsets.GenericViewSet):

    lookup_field = 'slug'
    queryset = Article.objects.select_related('author', 'author__user')
    permission_classes = (IsAuthenticatedOrReadOnly,)
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    # … Stuff

    def list(self, request):
        serializer_context = {'request': request}
-        serializer_instances = self.queryset.all()
+        page = self.paginate_queryset(self.queryset)

        serializer = self.serializer_class(
-            serializer_instances,
+            page,
            context=serializer_context,
            many=True
        )

-        return Response(serializer.data, status=status.HTTP_200_OK)
+        return self.get_paginated_response(serializer.data)
```

{x: pagination add pagination to articleviewset list}
Add pagination to the `.list()` method of `ArticleViewSet`.

Now our `.list()` endpoint will paginate properly. As a final measure, open Postman and send the “All Articles” request. If the response is successful, you should now see the data under the key `articles` and the number of articles found under the key `articlesCount`. Here “the number of articles found” means the total number of articles found in the database. Not just the number returned. This is so the client can figure out when to stop asking for more data — that is, when `page * limit > articlesCount`.