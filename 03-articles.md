# Articles

Right now, users can register, log in, and look at each other’s profile pages. Doesn’t sound very fun, does it? In this chapter, we’re going to create a system for posting new articles and having other users read those articles.

## Creating the Article model

I’m sure you’re starting to notice a pattern at this point. The first thing we do in each chapter is make a new model. This chapter is no different.

Create `conduit/apps/articles/models.py` and add the following content:

```python
from django.db import models

from conduit.apps.core.models import TimestampedModel


class Article(TimestampedModel):
    slug = models.SlugField(db_index=True, max_length=255, unique=True)
    title = models.CharField(db_index=True, max_length=255)

    description = models.TextField()
    body = models.TextField()

    # Every article must have an author. This will answer questions like "Who
    # gets credit for writing this article?" and "Who can edit this article?".
    # Unlike the `User` <-> `Profile` relationship, this is a simple foreign
    # key (or one-to-many) relationship. In this case, one `Profile` can have
    # many `Article`s.
    author = models.ForeignKey(
        'profiles.Profile', on_delete=models.CASCADE, related_name='articles'
    )

    def __str__(self):
        return self.title
```

With the `Article` model in place, we need to create and apply some more migrations.

To create migrations for the `articles` app, run the following:

```bash
~ python manage.py makemigrations articles
```

To apply the migrations, run the following:

```bash
~ python manage.py migrate
```

## Serializing articles

The second step in our process is to create a way to serialize the model instances we will be working with.

Create `conduit/apps/articles/serializers.py` and give it the following content:

```python
from rest_framework import serializers

from conduit.apps.profiles.serializers import ProfileSerializer

from .models import Article


class ArticleSerializer(serializers.ModelSerializer):
    author = ProfileSerializer(read_only=True)
    description = serializers.CharField(required=False)
    slug = serializers.SlugField(required=False)

    # Django REST Framework makes it possible to create a read-only field that
    # gets it's value by calling a function. In this case, the client expects
    # `created_at` to be called `createdAt` and `updated_at` to be `updatedAt`.
    # `serializers.SerializerMethodField` is a good way to avoid having the
    # requirements of the client leak into our API.
    createdAt = serializers.SerializerMethodField(method_name='get_created_at')
    updatedAt = serializers.SerializerMethodField(method_name='get_updated_at')

    class Meta:
        model = Article
        fields = (
            'author',
            'body',
            'createdAt',
            'description',
            'slug',
            'title',
            'updatedAt',
        )

    def create(self, validated_data):
        author = self.context.get('author', None)

        return Article.objects.create(author=author, **validated_data)

    def get_created_at(self, instance):
        return instance.created_at.isoformat()

    def get_updated_at(self, instance):
        return instance.updated_at.isoformat()
```

## Rendering articles

Now we can add a renderer like the ones we created in the past.

Create `conduit/apps/articles/renderers.py` with the following:

```python
from conduit.apps.core.renderers import ConduitJSONRenderer


class ArticleJSONRenderer(ConduitJSONRenderer):
    object_label = 'article'
```

## Creating articles

We’re going to add more views for articles than we have for other models, but let’s start by just making a view for creating articles.

Create `conduit/apps/articles/views.py` and use the following:

```python
from rest_framework import mixins, status, viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from rest_framework.response import Response

from .models import Article
from .renderers import ArticleJSONRenderer
from .serializers import ArticleSerializer


class ArticleViewSet(mixins.CreateModelMixin,
                     viewsets.GenericViewSet):

    queryset = Article.objects.select_related(‘author’, ‘author__user’)
    permission_classes = (IsAuthenticatedOrReadOnly,)
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    def create(self, request):
        serializer_context = {'author': request.user.profile}
        serializer_data = request.data.get('article', {})

        serializer = self.serializer_class(
            data=serializer_data, context=serializer_context
        )
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

Now let’s expose this endpoint to the web by adding it to our url configuration.

Create `conduit/apps/articles/urls.py` and give it the following content:

```python
from django.conf.urls import include, url

from rest_framework.routers import DefaultRouter

from .views import ArticleViewSet

router = DefaultRouter(trailing_slash=False)
router.register(r'articles', ArticleViewSet)

urlpatterns = [
    url(r'^', include(router.urls)),
]
```

Note the use of `trailing_slash=False` when creating the router. This is important because, by default, Django expects routes to have a trailing slash and our client does not follow this convention. Without `trailing_slash=False`, we could run into some issues with routes appearing to be defined, but the client recieving a 404 error when accessing those routes.

Now open up `conduit/urls.py` and apply the following changes:

```diff
urlpatterns = [
    url(r'^admin/', admin.site.urls),

+    url(r'^api/', include('conduit.apps.articles.urls', namespace='articles')),
    url(r'^api/', include('conduit.apps.authentication.urls', namespace='authentication')),
    url(r'^api/', include('conduit.apps.profiles.urls', namespace='profiles')),
]

```

You should be able to use Postman to create your first article at this point. Open up Postman and send the “Create Article” request in the “Articles” folder. You should get back an article, but you may notice that your article doesn’t have a slug. This is our next order of business.

## Slugifying article titles

In a community site like ours, it is possible that two users will create articles with the same title. Since slugifying a string only lowercases the letters and replaces spaces with hyphens, we need to do something a little special to avoid showing an error to someone posting a title that has been used before. Remember: our slugs must be unique.

Create `conduit/apps/core/utils.py` with the following:

```python
import random
import string

DEFAULT_CHAR_STRING = string.ascii_lowercase + string.digits

def generate_random_string(chars=DEFAULT_CHAR_STRING, size=6):
    return ''.join(random.choice(chars) for _ in range(size))
```

This short snippet simply generates a random string of length `size`.

We’re going to use a signal to automatically set a slug on any article that doesn’t have one before it’s saved. To do that, we first need to write our signal function.

Create `conduit/apps/articles/signals.py` and use the following code:

```python
from django.db.models.signals import pre_save
from django.dispatch import receiver
from django.utils.text import slugify

from conduit.apps.core.utils import generate_random_string

from .models import Article

@receiver(pre_save, sender=Article)
def add_slug_to_article_if_not_exists(sender, instance, *args, **kwargs):
    MAXIMUM_SLUG_LENGTH = 255

    if instance and not instance.slug:
        slug = slugify(instance.title)
        unique = generate_random_string()

        if len(slug) > MAXIMUM_SLUG_LENGTH:
            slug = slug[:MAXIMUM_SLUG_LENGTH]

        while len(slug + '-' + unique) > MAXIMUM_SLUG_LENGTH:
            parts = slug.split('-')

            if len(parts) is 1:
                # The slug has no hypens. To append the unique string we must
                # arbitrarly remove `len(unique)` characters from the end of
                # `slug`. Subtract one to account for extra hyphen.
                slug = slug[:MAXIMUM_SLUG_LENGTH - len(unique) - 1]
            else:
                slug = '-'.join(parts[:-1])

        instance.slug = slug + '-' + unique
```

Finally, we have to register a custom app config with Django for the articles app. This works just like the one for the authentication app.

Open `conduit/apps/articles/__init__.py` and write out the following:

```python
from django.apps import AppConfig


class ArticlesAppConfig(AppConfig):
    name = 'conduit.apps.articles'
    label = 'articles'
    verbose_name = 'Articles'

    def ready(self):
        import conduit.apps.articles.signals

default_app_config = 'conduit.apps.articles.ArticlesAppConfig'
```

Try using Postman to send the “Create Article” request again. This time, you should notice that the article has a slug.

## Listing and retrieving articles

Adding the ability to retrieve a single article or list all articles is simple at this point. Because we’re already using a DRF viewset, all we need to do is add the appropriate mixins.

Open `conduit/apps/articles/views.py` and make the following changes:

```diff
class ArticleViewSet(mixins.CreateModelMixin,
+                     mixins.ListModelMixin,
+                     mixins.RetrieveModelMixin,
                     viewsets.GenericViewSet):

+    lookup_field = 'slug'
    queryset = Article.objects.select_related(‘author’, ‘author__user’)
    permission_classes = (IsAuthenticatedOrReadOnly,)
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    # …
```

See how easy that was? DRF handles the heavy lifting for us.

Open Postman and try sending the “All Articles” and “Single Article by Slug” requests.

In the case of the “All Articles” request, you’ll notice that the API throws an error. No, this isn’t a mistake. This is what should happen.

Up until now, we have been assuming that `ConduitJSONRenderer` will receive a dictionary. In the case of rendering a list of objects, that’s not a valid assumption. When rendering a list, `data` is an instance of `ReturnList`. We need to handle this case.

Open `conduit/apps/core/renderers.py` and make the following changes:

```diff
import json

from rest_framework.renderers import JSONRenderer
from rest_framework.utils.serializer_helpers import ReturnList


class ConduitJSONRenderer(JSONRenderer):
    charset = 'utf-8'
    object_label = 'object'
+    object_label_plural = 'objects'

    def render(self, data, media_type=None, renderer_context=None):
+        if isinstance(data, ReturnList):
+            _data = json.loads(
+                super(ConduitJSONRenderer, self).render(data).decode('utf-8')
+            )

+            return json.dumps({
+                self.object_label_plural: _data
+            })
+        else:
            # If the view throws an error (such as the user can't be authenticated
            # or something similar), `data` will contain an `errors` key. We want
            # the default JSONRenderer to handle rendering errors, so we need to
            # check for this case.
            errors = data.get('errors', None)

            if errors is not None:
                # As mentioned about, we will let the default JSONRenderer handle
                # rendering errors.
                return super(ConduitJSONRenderer, self).render(data)

            return json.dumps({
                self.object_label: data
            })
```

A couple of things to note:

1. Notice how we indented the existing code that checks for the errors key. This indentation is important.
2. We have added another property to `ConduitJSONRenderer` called `object_label_plural`. When we’re rendering a list, we want the label to be the plural “users” instead of the singlular “user”.
3. We delegate rendering a list of objects to the default `JSONRenderer`.

Before we try the request again, let’s add `object_label_plural` to `ArticleJSONRenderer`.

Open `conduit/apps/articles/renderers.py` and make the following change:

```diff
class ArticleJSONRenderer(ConduitJSONRenderer):
    object_label = 'article'
+    object_label_plural = 'articles'
```

Once again, open Postman and make the “All Articles” request. The error will be gone and the results will be formatted nicely.

## Updating an article

Now that we can create, list, and retrieve articles, let’s work on adding the ability to update an article.

Open `conduit/apps/articles/views.py` and make the following changes:

```diff
from rest_framework import mixins, status, viewsets
+from rest_framework.exceptions import NotFound
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from rest_framework.response import Response

from .models import Article
from .renderers import ArticleJSONRenderer
from .serializers import ArticleSerializer

class ArticleViewSet(mixins.CreateModelMixin,
                     mixins.ListModelMixin,
                     mixins.RetrieveModelMixin,
                     viewsets.GenericViewSet):

    lookup_field = 'slug'
    queryset = Article.objects.select_related('author', 'author__user')
    permission_classes = (IsAuthenticatedOrReadOnly,)
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    # Create …

+    def update(self, request, slug):
+        try:
+            serializer_instance = self.queryset.get(slug=slug)
+        except Article.DoesNotExist:
+            raise NotFound('An article with this slug does not exist.')
+            
+        serializer_data = request.data.get('article', {})
+
+        serializer = self.serializer_class(
+            serializer_instance, data=serializer_data, partial=True
+        )
+        serializer.is_valid(raise_exception=True)
+        serializer.save()
+
+        return Response(serializer.data, status=status.HTTP_200_OK)
```

One thing we haven’t had to do yet is handle the case where the object being requested doesn’t exist. The retrieve function has to handle this case too, but we just used DRF’s default retrieve function for now. We will clean that up in a minute, but first let’s use Postman to test our update endpoint.

Before we move on, it’s worth pointing out that we’re raising a new type of exception that we haven’t handled before. That means we need to add it to our custom exception handler.

Open `conduit/apps/core/exceptions.py` and make these changes:

```diff
def core_exception_handler(exc, context):
    # If an exception is thrown that we don't explicitly handle here, we want
    # to delegate to the default exception handler offered by DRF. If we do
    # handle this exception type, we will still want access to the response
    # generated by DRF, so we get that response up front.
    response = exception_handler(exc, context)
    handlers = {
+        'NotFound': _handle_not_found_error,
        'ValidationError': _handle_generic_error
    }
    # This is how we identify the type of the current exception. We will use
    # this in a moment to see whether we should handle this exception or let
    # Django REST Framework do it's thing.
    exception_class = exc.__class__.__name__

    if exception_class in handlers:
        # If this exception is one that we can handle, handle it. Otherwise,
        # return the response generated earlier by the default exception 
        # handler.
        return handlers[exception_class](exc, context, response)

    return response

# handle_generic_error …

+def _handle_not_found_error(exc, context, response):
+    view = context.get('view', None)
+
+    if view and hasattr(view, 'queryset') and view.queryset is not None:
+        error_key = view.queryset.model._meta.verbose_name
+
+        response.data = {
+            'errors': {
+                error_key: response.data['detail']
+            }
+        }
+
+    else:
+        response = _handle_generic_error(exc, context, response)
+
+    return response
```

One important thing to point out here is that, for `_handle_not_found_error` to work, the view must have specified a `queryset` property. We have done this so far, so it’s no problem for us.

Open Postman and send the “Update Article” request using a valid article slug. Your changes should take effect.

Just for completeness, send another Postman “Update Article” request. This time you should use a slug that doesn’t exist. You should see the error message we wrote above.

Since we’re dealing with errors anyways, let’s get rid of that `ProfileNotFoundError` exception we created earlier and replace it with `NotFound`.

Open `conduit/apps/profiles/views.py` and make this change:

```diff
from rest_framework import status
+from rest_framework.exceptions import NotFound
from rest_framework.generics import RetrieveAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

-from .exceptions import ProfileDoesNotExist
from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer


class ProfileRetrieveAPIView(RetrieveAPIView):
    permission_classes = (AllowAny,)
+    queryset = Profile.objects.select_related(‘user’)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def retrieve(self, request, username, *args, **kwargs):
        # Try to retrieve the requested profile and throw an exception if the
        # profile could not be found.
        try:
-            # We use the `select_related` method to avoid making unnecessary
-            # database calls.
-            profile = Profile.objects.select_related('user').get(
-                user__username=username
-            )
+            profile = self.queryset.get(user__username=username)
        except Profile.DoesNotExist:
-            raise ProfileDoesNotExist
+            raise NotFound(‘A profile with this username does not exist.’)

        serializer = self.serializer_class(profile)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

You may want to take this opportunity to send the “Profile” request in the “Profiles” Postman folder to verify this change worked. Send a request for an invalid username to test the error.

At this point, you’ll also want to delete `conduit/apps/profiles/exceptions.py` and remove `ProfileDoesNotExist` from the list of error handlers in `conduit/apps/core/exceptions.py`.

Now let’s go back and write our own `retrieve` function so that our errors come out right. Head over to `conduit/apps/articles/views.py` and make this change:

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

    # Create …

    def retrieve(self, request, slug):
        try:
            serializer_instance = self.queryset.get(slug=slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug does not exist.')

        serializer = self.serializer_class(serializer_instance)

        return Response(serializer.data, status=status.HTTP_200_OK)

    # Update …
```

Once again, open Postman and send the “Single Article by Slug” request with an invalid slug. You should see the error from the above snippet, formatted properly.

## I have a comment to make …

In the next chapter we will tackle one of the more social aspects of our app: commenting. Users will be able to comment on articles they like and have discussions in those comments with other users.