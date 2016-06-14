# Favoriting

We talked before about having a way to tell other users which articles you liked and a way to save those articles so you can easily find them later. Favoriting is what we were talking about. In this chapter we will add the ability to favorite an article by updating both the profile model and the article serializer and adding new endpoints for favoriting and unfavoriting articles.

## Updating the Profile model

Let’s open `conduit/apps/profiles/models.py`. We need to add a many-to-many relationship between `Profile` and `Article` and add some utility methods similar to what we did for following relationships.

Make the following changes to `Profile`:

```diff
class Profile(TimestampedModel):
    # … Properties

    # This is an example of a Many-To-Many relationship where both sides of the
    # relationship are of the same model. In this case, the model is `Profile`.
    # As mentioned in the text, this relationship will be one-way. Just because
    # you are following mean does not mean that I am following you. This is
    # what `symmetrical=False` does for us.
    follows = models.ManyToManyField(
        'self',
        related_name='followed_by',
        symmetrical=False
    )

+    favorites = models.ManyToManyField(
+        'articles.Article',
+        related_name='favorited_by'
+    )
+
    def __str__(self):
        return self.user.username

    # … Follow methods

+    def favorite(self, article):
+        """Favorite `article` if we haven't already favorited it."""
+        self.favorites.add(article)
+
+    def unfavorite(self, article):
+        """Unfavorite `article` if we've already favorited it."""
+        self.favorites.remove(article)
+
+    def has_favorited(self, article):
+        """Returns True if we have favorited `article`; else False."""
+        return self.favorites.filter(pk=article.pk).exists()
```

Make and apply migrations for these changes by running the normal commands:

```bash
~ python manage.py makemigrations
```

```bash
~ python manage.py migrate
```

## Updating ArticleSerializer

Open `conduit/apps/articles/serializers.py` and apply these changes:

```diff
class ArticleSerializer(serializers.ModelSerializer):
    author = ProfileSerializer(read_only=True)
    description = serializers.CharField(required=False)
    slug = serializers.SlugField(required=False)

+    favorited = serializers.SerializerMethodField()
+    favoritesCount = serializers.SerializerMethodField(
+        method_name='get_favorites_count'
+    )

    createdAt = serializers.SerializerMethodField(method_name='get_created_at')
    updatedAt = serializers.SerializerMethodField(method_name='get_updated_at')

    class Meta:
        model = Article
        fields = (
            'author',
            'body',
            'createdAt',
            'description',
+            'favorited',
+            'favoritesCount',
            'slug',
            'title',
            'updatedAt',
        )

    def create(self, validated_data):
        author = self.context.get('author', None)

        return Article.objects.create(author=author, **validated_data)

    def get_created_at(self, instance):
        return instance.created_at.isoformat()

+    def get_favorited(self, instance):
+        request = self.context.get('request', None)
+
+        if request is None:
+            return False
+
+        if not request.user.is_authenticated():
+            return False
+
+        return request.user.profile.has_favorited(instance)
+
+    def get_favorites_count(self, instance):
+        return instance.favorited_by.count()

    def get_updated_at(self, instance):
        return instance.updated_at.isoformat()
```

## Updating ArticleViewSet

Notice that the serializer now requires the current request be passed as context to make sure we show the correct boolean value for `favorited`. Let’s make the changes to `ArticleViewSet` to let that happen.

Open `conduit/apps/articles/views.py`. There are a lot of changes this time.

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

    def create(self, request):
-        serializer_context = {'author': request.user.profile}
+        serializer_context = {
+            'author': request.user.profile,
+            'request': request
+        }
        serializer_data = request.data.get('article', {})

        serializer = self.serializer_class(
            data=serializer_data, context=serializer_context
        )
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_201_CREATED)

+    def list(self, request):
+        serializer_context = {'request': request}
+        serializer_instances = self.queryset.all()
+
+        serializer = self.serializer_class(
+            serializer_instances,
+            context=serializer_context,
+            many=True
+        )
+
+        return Response(serializer.data, status=status.HTTP_200_OK)

    def retrieve(self, request, slug):
+        serializer_context = {'request': request}
        try:
            serializer_instance = self.queryset.get(slug=slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug does not exist.')

-        serializer = self.serializer_class(serializer_instance)
+        serializer = self.serializer_class(
+            serializer_instance, 
+            context=serializer_context
+        )

        return Response(serializer.data, status=status.HTTP_200_OK)

    def update(self, request, slug):
+        serializer_context = {'request': request}
        try:
            serializer_instance = self.queryset.get(slug=slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug does not exist.')
            
        serializer_data = request.data.get('article', {})

-        serializer = self.serializer_class(
-            serializer_instance, data=serializer_data, partial=True
-        )
+        serializer = self.serializer_class(
+            serializer_instance, 
+            context=serializer_context,
+            data=serializer_data, 
+            partial=True
+        )
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_200_OK)
```

All we’re doing above is passing the request as context to the serializer for each method. Since we need to pass the context to the serializer, we can’t rely on the default `.list()` method anymore, so we created our own.

## Retrieving an Article with Postman

Now is a good time to use Postman to get a list of articles. This will make sure we haven’t broken anything and it will also point out what we need to do next.

After you send the Postman “All Articles” request, you will notice that the response includes `”following”: false`. This is correct because we haven’t favorited any of the articles yet. We need to add new endpoints for that.

## Adding favoriting and unfavoriting endpoints

Open `conduit/apps/articles/views.py` and apply the following changes to the import statements:

```diff
from rest_framework import generics, mixins, status, viewsets
from rest_framework.exceptions import NotFound
-from rest_framework import IsAuthenticatedOrReadOnly
+from rest_framework.permissions import (
+    IsAuthenticated, IsAuthenticatedOrReadOnly
+)
from rest_framework.response import Response
+from rest_framework.views import APIView

from .models import Article, Comment
from .renderers import ArticleJSONRenderer, CommentJSONRenderer
from .serializers import ArticleSerializer, CommentSerializer
```

Now we can add our new view:

```python
class ArticlesFavoriteAPIView(APIView):
    permission_classes = (IsAuthenticated,)
    renderer_classes = (ArticleJSONRenderer,)
    serializer_class = ArticleSerializer

    def delete(self, request, article_slug=None):
        profile = self.request.user.profile
        serializer_context = {'request': request}

        try:
            article = Article.objects.get(slug=article_slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug was not found.')

        profile.unfavorite(article)

        serializer = self.serializer_class(article, context=serializer_context)

        return Response(serializer.data, status=status.HTTP_200_OK)

    def post(self, request, article_slug=None):
        profile = self.request.user.profile
        serializer_context = {'request': request}

        try:
            article = Article.objects.get(slug=article_slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug was not found.')

        profile.favorite(article)

        serializer = self.serializer_class(article, context=serializer_context)

        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

This view is straightfoward. Either we favorite an article or we unfavorite it. We throw an exception if the article doesn’t exist and we respond with a serialized version of the article.

Open `conduit/apps/articles/urls.py` and add this new URL:

```diff
from django.conf.urls import include, url

from rest_framework.routers import DefaultRouter

from .views import (
-    ArticleViewSet, CommentsListCreateAPIView, CommentsDestroyAPIView
+    ArticleViewSet, ArticlesFavoriteAPIView, CommentsListCreateAPIView,
+    CommentsDestroyAPIView
)

router = DefaultRouter(trailing_slash=False)
router.register(r'articles', ArticleViewSet)

urlpatterns = [
    url(r'^', include(router.urls)),

+    url(r'^articles/(?P<article_slug>[-\w]+)/favorite/?$',
+        ArticlesFavoriteAPIView.as_view()),
+
    url(r'^articles/(?P<article_slug>[-\w]+)/comments/?$', 
        CommentsListCreateAPIView.as_view()),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/(?P<comment_pk>[\d]+)/?$',
        CommentsDestroyAPIView.as_view()),
]
```

## Favoriting and unfavoriting articles with Postman

Once again, open up Postman. This time we will be sending the “Favorite Article” and “Unfavorite Article” requests. If these requests are successful, you will get the serialized article as the response. For the “Favorite Article” request, this should include `”favorited”: true,`. For the “Unfavorite Article” request, the response should include `”favorited”: false,`.