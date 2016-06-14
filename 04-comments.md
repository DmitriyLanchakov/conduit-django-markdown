# Comments

I bet you’re getting bored by now, huh? A lot of this stuff is the same as previous chapters. First we’ll create a model, then a serializer, followed by a renderer, and eventually a set of views.

## Creating the Comment model

Open `conduit/apps/articles/models.py` and add the following model to the end of the file:

```python
class Comment(TimestampedModel):
    body = models.TextField()

    article = models.ForeignKey(
        'articles.Article', related_name='comments', on_delete=models.CASCADE
    )

    author = models.ForeignKey(
        'profiles.Profile', related_name='comments', on_delete=models.CASCADE
    )
```

The `Comment` model is very simple. Aside from having an `author` and being related to a specific `article`, all this data model has is a text field called `body` for storing the content of the comment.

The next step would be to create and run your migrations as usual. 

To generate new migrations, run this command:

```bash
~ python manage.py makemigrations
```

To apply those new migrations, run this command:

```bash
~ python manage.py migrate
```

## Serializing the Comment model

Open `conduit/apps/articles/serializers.py`. Make this change to the imports:

```diff
from rest_framework import serializers

from conduit.apps.profiles.serializers import ProfileSerializer

-from .models import Article
+from .models import Article, Comment
```

Then add the new serializer to the end of the file:

```python
class CommentSerializer(serializers.ModelSerializer):
    author = ProfileSerializer(required=False)

    createdAt = serializers.SerializerMethodField(method_name='get_created_at')
    updatedAt = serializers.SerializerMethodField(method_name='get_updated_at')

    class Meta:
        model = Comment
        fields = (
            'id',
            'author',
            'body',
            'createdAt',
            'updatedAt',
        )

    def create(self, validated_data):
        article = self.context['article']
        author = self.context['author']

        return Comment.objects.create(
            author=author, article=article, **validated_data
        )

    def get_created_at(self, instance):
        return instance.created_at.isoformat()

    def get_updated_at(self, instance):
        return instance.updated_at.isoformat()
```

This is all stuff we’ve seen before, so we want waste any breath discussing it.

## Rendering a Comment

Comments don’t require any special treatment for rendering, so we’ll do it just like we have in the past.

Open `conduit/apps/articles/renderers.py` and add the following to the bottom of the file:

```python
class CommentJSONRenderer(ConduitJSONRenderer):
    object_label = 'comment'
    object_label_plural = 'comments'
```

## Listing and creating comments

To round off our model-serializer-renderer-view ritual, let’s add some views for comments.

Open `conduit/apps/articles/views.py` and make these changes to the import statements:

```diff
from rest_framework import generics, mixins, status, viewsets
from rest_framework.exceptions import NotFound
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from rest_framework.response import Response

-from .models import Article
+from .models import Article, Comment
-from .renderers import ArticleJSONRenderer
+from .renderers import ArticleJSONRenderer, CommentJSONRenderer
-from .serializers import ArticleSerializer
+from .serializers import ArticleSerializer, CommentSerializer
```

Then add this new view to the bottom of the file:

```python
class CommentsListCreateAPIView(generics.ListCreateAPIView):
    lookup_field = 'article__slug'
    lookup_url_kwarg = 'article_slug'
    permission_classes = (IsAuthenticatedOrReadOnly,)
    queryset = Comment.objects.select_related(
        'article', 'article__author', 'article__author__user',
        'author', 'author__user'
    )
    renderer_classes = (CommentJSONRenderer,)
    serializer_class = CommentSerializer

    def filter_queryset(self, queryset):
        # The built-in list function calls `filter_queryset`. Since we only
        # want comments for a specific article, this is a good place to do
        # that filtering.
        filters = {self.lookup_field: self.kwargs[self.lookup_url_kwarg]}

        return queryset.filter(**filters)

    def create(self, request, article_slug=None):
        data = request.data.get('comment', {})
        context = {'author': request.user.profile}

        try:
            context['article'] = Article.objects.get(slug=article_slug)
        except Article.DoesNotExist:
            raise NotFound('An article with this slug does not exist.')

        serializer = self.serializer_class(data=data, context=context)
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

Now open up `conduit/apps/articles/urls.py` and make the following change to the import statements:

```diff
-from .views import ArticleViewSet
+from .views import (
+    ArticleViewSet, CommentsListCreateAPIView, CommentsDestroyAPIView
+)
```

And make these changes to `urlpatterns`:

```diff
urlpatterns = [
    url(r'^', include(router.urls)),
+
+    url(r'^articles/(?P<article_slug>[-\w]+)/comments/?$', 
+        CommentsListCreateAPIView.as_view()),
]
```

## Creating and listing comments with Postman

This is a good opportunity to stop and test out our new endpoints.

Open up Postman and send the “Create Comment for Article” request in the “Comments” folder. You should receive the new comment in the response.

Now send the “All Comments for Article” request and you should see the comment you just created.

## Destroying comments

The next task on the our to-do list is adding an endpoint for destroying comments.

Open up `conduit/apps/articles/views.py` and add the following:

```python
class CommentsDestroyAPIView(generics.DestroyAPIView):
    lookup_url_kwarg = 'comment_pk'
    permission_classes = (IsAuthenticatedOrReadOnly,)
    queryset = Comment.objects.all()

    def destroy(self, request, article_slug=None, comment_pk=None):
        try:
            comment = Comment.objects.get(pk=comment_pk)
        except Comment.DoesNotExist:
            raise NotFound('A comment with this ID does not exist.')

        comment.delete()

        return Response(None, status=status.HTTP_204_NO_CONTENT)
```

As always, we need to add this to our `urlpatterns`. Open `conduit/apps/articles/urls.py` and apply the following changes:

```diff
from django.conf.urls import include, url

from rest_framework.routers import DefaultRouter

-from .views import ArticleViewSet, CommentsListCreateAPIView
+from .views import (
+    ArticleViewSet, CommentsListCreateAPIView, CommentsDestroyAPIView
+)

router = DefaultRouter(trailing_slash=False)
router.register(r'articles', ArticleViewSet)

urlpatterns = [
    url(r'^', include(router.urls)),

    url(r'^articles/(?P<article_slug>[-\w]+)/comments/?$', 
        CommentsListCreateAPIView.as_view()),
+
+    url(r'^articles/(?P<article_slug>[-\w]+)/comments/(?P<comment_pk>[\d]+)/?$',
+        CommentsDestroyAPIView.as_view()),
]
```

## Destroying comments with Postman

Open up Postman and send the “Delete Comment for Article” request from the “Comments” folder.

If the request is successful, you should get an empty response with a `204 No Content` status code.