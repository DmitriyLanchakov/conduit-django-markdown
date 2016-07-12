# Filtering

Filtering is about allowing the user to be more specific about what they want to see. In this chapter we will add a new endpoint for seeing articles posted by anyone the current user follows. We will also add filtering to the articles list endpoint. Users will be able to filters articles by the author, by tag, and by whether they’ve favorited the articles.

This is the last chapter. We’re almost done, but we still need to power through, so let’s do it.

## Creating a feed endpoint

To get started, we’re going to create a new endpoint for seeing articles posted by anyone the current user follows. We will call this the current user’s “Feed”.

Open `conduit/apps/articles/views.py` and add this new class:

```python
class ArticlesFeedAPIView(generics.ListAPIView):
    permission_classes = (IsAuthenticated,)
    queryset = Article.objects.all()
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    def get_queryset(self):
        return Article.objects.filter(
            author__in=self.request.user.profile.follows.all()
        )

    def list(self, request):
        queryset = self.get_queryset()
        page = self.paginate_queryset(queryset)

        serializer_context = {'request': request}
        serializer = self.serializer_class(
            page, context=serializer_context, many=True
        )

        return self.get_paginated_response(serializer.data)
```

{x: filtering add articlesfeedapiview}
Create the `ArticlesFeedAPIView` view.

Looks pretty similar to what we’ve done before, right? The big change here is that we added a method called `.get_queryset()`. Why add this when we already have the `queryset` property set? Because, when setting the property, we don’t have access to the `request`. We need this to filter out authors the current user isn’t following.

Open `conduit/apps/articles/urls.py` and make this change:

```diff
from django.conf.urls import include, url

from rest_framework.routers import DefaultRouter

from .views import (
-    ArticleViewSet, ArticlesFavoriteAPIView, CommentsListCreateAPIView,
-    CommentsDestroyAPIView, TagListAPIView,
+    ArticleViewSet, ArticlesFavoriteAPIView, ArticlesFeedAPIView,
+    CommentsListCreateAPIView, CommentsDestroyAPIView, TagListAPIView
)

router = DefaultRouter(trailing_slash=False)
router.register(r'articles', ArticleViewSet)

urlpatterns = [
+    url(r'^articles/feed/?$', ArticlesFeedAPIView.as_view()),
+
    url(r'^articles/(?P<article_slug>[-\w]+)/favorite/?$',
        ArticlesFavoriteAPIView.as_view()),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/?$', 
        CommentsListCreateAPIView.as_view()),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/(?P<comment_pk>[\d]+)/?$',
        CommentsDestroyAPIView.as_view()),

    url(r'^tags/?$', TagListAPIView.as_view()),

    url(r'^', include(router.urls)),
]
```

{x: filtering add articlesfeedapiview to urlpatterns}
Add `ArticlesFeedAPIView` to `urlpatterns`.

It is important that we add the URL for the feed view to the top of `urlpatterns`. If we don’t then `articles/feed/` will match the URL for each of the URLS for favoriting articles, list and creating comments, and destroying comments.

You should open Postman and send the “Feed” request in the “Articles” folder. The response will depend on whether you’ve followed other users and can range from including no articles to including all of the articles you’ve created. To verify this works, try creating a new user and hitting the feed endpoint. It should return no results. Sending the “All Articles” request should still return all of the articles in the database.

## Filtering by author, favorited, and tags

Now we’re going to add a few filters to the articles list view. 

Open `conduit/apps/articles/views.py` and make these changes:

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

+    def get_queryset(self):
+        queryset = self.queryset
+
+        author = self.request.query_params.get('author', None)
+        if author is not None:
+            queryset = queryset.filter(author__user__username=author)
+
+        tag = self.request.query_params.get('tag', None)
+        if tag is not None:
+            queryset = queryset.filter(tags__tag=tag)
+
+        favorited_by = self.request.query_params.get('favorited', None)
+        if favorited_by is not None:
+            queryset = queryset.filter(
+                favorited_by__user__username=favorited_by
+            )
+
+        return queryset

    # … Create

    def list(self, request):
        serializer_context = {'request': request}
-        page = self.paginate_queryset(self.queryset)
+        page = self.paginate_queryset(self.get_queryset())

        serializer = self.serializer_class(
            page,
            context=serializer_context,
            many=True
        )

        return self.get_paginated_response(serializer.data)
```

{x: filtering add get_queryset}
Add the `.get_queryset()` method to `ArticleViewSet` and call it from the `.list()` method.

With these changes made, you should be able to open up Postman and send each of the “Articles by Author”, “Articles Favorited by Username”, and “Articles by Tag” requests and get successful responses. Like the feed endpoint, the responses will vary depending on factors like which user wrote articles, who favorited what, and which article has what tags. The easiest way to know if these filters work is to make sure they aren’t broken. The filters are probably broken if the three requests above all return the same results as the “All Articles” request, but this depends on the same factors as above.