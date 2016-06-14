# Tagging

In our quest to help users find what they’re looking for, we’re going to make it possible to tag articles. If you’re interested in articles on Django, for example, you should have a way to quickly find all articles on Django. Tagging will give us that.

## Creating the Tag model

Open up `conduit/apps/articles/models.py` and add the new `Tag` model, like so:

```python
class Tag(TimestampedModel):
    tag = models.CharField(max_length=255)
    slug = models.SlugField(db_index=True, unique=True)

    def __str__(self):
        return self.tag
```

The `Tag` model is one of the most simple models you can create. There is a lot more we can do with tags, but this simple model will suffice for our MVP.

Now apply this change to the `Article` model:

```diff
class Article(TimestampedModel):
    slug = models.SlugField(db_index=True, max_length=255, unique=True)
    title = models.CharField(db_index=True, max_length=255)

    description = models.TextField()
    body = models.TextField()

    author = models.ForeignKey(
        'profiles.Profile', on_delete=models.CASCADE, related_name='articles'
    )

+    tags = models.ManyToManyField(
+        'articles.Tag', related_name='articles'
+    )

    def __str__(self):
        return self.title
```

Now that tags are related to articles, we can create and apply our migrations:

```bash
~ python manage.py makemigrations
```

```bash
~ python manage.py migrate
```

## Serializing Tag objects

We are going to treat tags a little differently when serializing them. When we serializer other objects like users or articles, we want to namespace them. This is not the case with tags. For tags, we want an array of strings. 

Another thing that is different about tags is that, if a user tags an article with a tag that doesn’t exist, we should create it. Normally we would throw an exception in this case, but it would create a bad user experience if users had to create tags before they could use them.

To accomplish these goals, we’re going to create a special serializer field for tags.

Create `conduit/apps/articles/relations.py` and give it the following content:

```python
from rest_framework import serializers

from .models import Tag


class TagRelatedField(serializers.RelatedField):
    def get_queryset(self):
        return Tag.objects.all()

    def to_internal_value(self, data):
        tag, created = Tag.objects.get_or_create(tag=data, slug=data.lower())

        return tag

    def to_representation(self, value):
        return value.tag
```

Notice that `to_internal_value` uses the `get_or_create()` method to do what we described above. `to_representation` simply returns the `tag` property of the tag object.

## Serializing Articles with Tags

The next step is to use the special field we just created to serialize articles with tags.

Open `conduit/apps/articles/serializers.py` and add this new import:

```diff
from rest_framework import serializers

from conduit.apps.profiles.serializers import ProfileSerializer

from .models import Article, Comment
+from .relations import TagRelatedField
```

We can then make the following changes to `ArticleSerializer`:

```diff
class ArticleSerializer(serializers.ModelSerializer):
    author = ProfileSerializer(read_only=True)
    description = serializers.CharField(required=False)
    slug = serializers.SlugField(required=False)

    favorited = serializers.SerializerMethodField()
    favoritesCount = serializers.SerializerMethodField(
        method_name='get_favorites_count'
    )

+    tagList = TagRelatedField(many=True, required=False, source='tags')
+
    createdAt = serializers.SerializerMethodField(method_name='get_created_at')
    updatedAt = serializers.SerializerMethodField(method_name='get_updated_at')

    class Meta:
        model = Article
        fields = (
            'author',
            'body',
            'createdAt',
            'description',
            'favorited',
            'favoritesCount',
            'slug',
+            'tagList',
            'title',
            'updatedAt',
        )

        def create(self, validated_data):
        author = self.context.get('author', None)

        return Article.objects.create(author=author, **validated_data)

    def create(self, validated_data):
        author = self.context.get('author', None)
+        tags = validated_data.pop('tags', [])
+
+        article = Article.objects.create(author=author, **validated_data)
+
+        for tag in tags:
+            article.tags.add(tag)

-        return Article.objects.create(author=author, **validated_data)
+        return article

    # … Other methods
```

## Tagging Articles with Postman

The first thing we want to do is retrieve an article that has no tags. This is easy because none of our articles have tags at the moment. Send the “All Articles” request and notice how the response now includes `”tagList”: [],`.

Create a new article with tags by sending the “Create Article” request. The response should include `”tagList”: [“training”, “dragons”],`.

## Serializing Tag objects

We also want to be able to retrieve a list of all the tags that have ever been used. First, we need a `TagSerializer`.

Open up `conduit/apps/articles/serializers.py` and make this change to the imports:

```diff
from rest_framework import serializers

from conduit.apps.profiles.serializers import ProfileSerializer

-from .models import Article, Comment
+from .models import Article, Comment, Tag
from .relations import TagRelatedField
```

Now add this new serializer:

```python
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ('tag',)

    def to_representation(self, obj):
        return obj.tag
```

Serializing tags is simple because the model is simple.

## Adding a Tag view

Open up `conduit/apps/articles/views.py` and apply these changes to the import statements:

```diff
from rest_framework import generics, mixins, status, viewsets
from rest_framework.exceptions import NotFound
from rest_framework.permissions import (
-    IsAuthenticated, IsAuthenticatedOrReadOnly
+    AllowAny, IsAuthenticated, IsAuthenticatedOrReadOnly
)
from rest_framework.response import Response
from rest_framework.views import APIView

-from .models import Article, Comment
+from .models import Article, Comment, Tag
from .renderers import ArticleJSONRenderer, CommentJSONRenderer
-from .serializers import ArticleSerializer, CommentSerializer
+from .serializers import ArticleSerializer, CommentSerializer, TagSerializer
```

At the bottom of the file, we can add our new tag view:

```python
class TagListAPIView(generics.ListAPIView):
    queryset = Tag.objects.all()
    pagination_class = None
    permission_classes = (AllowAny,)
    serializer_class = TagSerializer

    def list(self, request):
        serializer_data = self.get_queryset()
        serializer = self.serializer_class(serializer_data, many=True)

        return Response({
            'tags': serializer.data
        }, status=status.HTTP_200_OK)
```

Finally, let’s add a new route for this view. Open `conduit/apps/articles/urls.py` and make this change:

```diff
from django.conf.urls import include, url

from rest_framework.routers import DefaultRouter

from .views import (
    ArticleViewSet, ArticlesFavoriteAPIView, CommentsListCreateAPIView,
-    CommentsDestroyAPIView
+    CommentsDestroyAPIView, TagListAPIView
)

router = DefaultRouter(trailing_slash=False)
router.register(r'articles', ArticleViewSet)

urlpatterns = [
    url(r'^', include(router.urls)),

    url(r'^articles/(?P<article_slug>[-\w]+)/favorite/?$',
        ArticlesFavoriteAPIView.as_view()),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/?$', 
        CommentsListCreateAPIView.as_view()),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/(?P<comment_pk>[\d]+)/?$',
        CommentsDestroyAPIView.as_view()),
+
+    url(r'^tags/?$', TagListAPIView.as_view()),
]
```

## Listing all Tags with Postman

To verify our work from this chapter, open Postman and send the “All Tags” request in the “Tags” folder. If the request is successful, the response will look like this:

```json
{
    “tags”: [
        “training”,
        “dragons”
    ]
}
```

If you’ve created any extra tags of your own, they should show up too.