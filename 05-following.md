# Following

We’re going to add a way for users to follow eachother. Following will be one-way, so it will be more like following on Twitter and less like becoming friends on Facebook.

Let’s get started.

## Creating the Following relationship

Open up `conduit/apps/profiles/models.py` and make these changes to `Profile`:

```diff
class Profile(TimestampedModel):
    # … User, bio, image, etc

+    # This is an example of a Many-To-Many relationship where both sides of the
+    # relationship are of the same model. In this case, the model is `Profile`.
+    # As mentioned in the text, this relationship will be one-way. Just because
+    # you are following mean does not mean that I am following you. This is
+    # what `symmetrical=False` does for us.
+    follows = models.ManyToManyField(
+        'self',
+        related_name='followed_by',
+        symmetrical=False
+    )

    def __str__(self):
        return self.user.username

+    def follow(self, profile):
+        """Follow `profile` if we're not already following `profile`."""
+        self.follows.add(profile)
+
+    def unfollow(self, profile):
+        """Unfollow `profile` if we're already following `profile`."""
+        self.follows.remove(profile)
+
+    def is_following(self, profile):
+        """Returns True if we're following `profile`; False otherwise."""
+        return self.follows.filter(pk=profile.pk).exists()
+
+    def is_followed_by(self, profile):
+        """Returns True if `profile` is following us; False otherwise."""
+        return self.followed_by.filter(pk=profile.pk).exists()
```

These changes to the follow do a few things:

1. We set up the actual following relationship.
2. We add methods to make it easy to perform the following/unfollowing tasks and to answer questions like “Am I following this person?” and “Is this person following me?”.

Go ahead and make new migrations and run them with the following two commands:

```bash
~ python manage.py makemigrations
```

```bash
~ python manage.py migrate
```

## Serializing the Following relationship

Next you’ll need to open `conduit/apps/profiles/serializers.py` and apply these changes:

```diff
class ProfileSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source='user.username')
    bio = serializers.CharField(allow_blank=True, required=False)
    image = serializers.CharField(allow_blank=True, required=False)
+    following = serializers.SerializerMethodField()

    class Meta:
        model = Profile
-        fields = (‘username’, ‘bio’, ‘image’,)
+        fields = ('username', 'bio', 'image', 'following',)
        read_only_fields = ('username',)
+
+    def get_following(self, instance):
+        request = self.context.get('request', None)
+
+        if request is None:
+            return False
+
+        if not request.user.is_authenticated():
+            return False
+
+        follower = request.user.profile
+        followee = instance
+
+        return follower.is_following(followee)
```

## Updating ProfileRetrieveAPIView

Open `conduit/apps/profiles/views.py`. We’re going to add two new views: one for following and one for unfollowing, but we first need to make a small change to `ProfileRetrieveAPIView`:

```diff
class ProfileRetrieveAPIView(RetrieveAPIView):
    # … Properties

    def retrieve(self, request, username, *args, **kwargs):
        # Try to retrieve the requested profile and throw an exception if the
        # profile could not be found.
        try:
            profile = self.queryset.get(user__username=username)
        except Profile.DoesNotExist:
            raise NotFound('A profile with this username was not found.')

-        serializer = self.serializer_class(profile)
+        serializer = self.serializer_class(profile, context={
+            'request': request
+        })

        return Response(serializer.data, status=status.HTTP_200_OK)
```

In the changes we just made to the serializer, you may have noticed that we’re looking for the `request` object inside the serializer. This change to `ProfileRetrieveAPIView` is how we can do that.

## Creating views for Following and Unfollowing

With `conduit/apps/profiles/views.py` still open, make the following changes to the import statements:

```diff
-from rest_framework import status
+from rest_framework import serializers, status
from rest_framework.exceptions import NotFound
from rest_framework.generics import RetrieveAPIView
-from rest_framework.permissions import AllowAny
+from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
+ from rest_framework.views import APIView

from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer
```

And now we can create our new views by adding this to the bottom of the file:

```python
class ProfileFollowAPIView(APIView):
    permission_classes = (IsAuthenticated,)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def delete(self, request, username=None):
        follower = self.request.user.profile

        try:
            followee = Profile.objects.get(user__username=username)
        except Profile.DoesNotExist:
            raise NotFound('A profile with this username was not found.')

        follower.unfollow(followee)

        serializer = self.serializer_class(followee, context={
            'request': request
        })
        
        return Response(serializer.data, status=status.HTTP_200_OK)

    def post(self, request, username=None):
        follower = self.request.user.profile

        try:
            followee = Profile.objects.get(user__username=username)
        except Profile.DoesNotExist:
            raise NotFound('A profile with this username was not found.')

        if follower.pk is followee.pk:
            raise serializers.ValidationError('You can not follow yourself.')

        follower.follow(followee)

        serializer = self.serializer_class(followee, context={
            'request': request
        })

        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

This should all look similar to what we’ve seen in the past. Now let’s add our new URLs and we can move on.

Open `conduit/apps/profiles/urls.py` and apply these changes:

```diff
from django.conf.urls import url

-from .views import ProfileRetrieveAPIView
+from .views import ProfileRetrieveAPIView, ProfileFollowAPIView

urlpatterns = [
    url(r'^profiles/(?P<username>\w+)/?$', ProfileRetrieveAPIView.as_view()),
+    url(r'^profiles/(?P<username>\w+)/follow/?$', 
+        ProfileFollowAPIView.as_view()),
]
```

## Following and Unfollowing with Postman

Crack open Postman and send the “Follow Profile” and “Unfollow Profile” requests in the “Profiles” folder. If the requests are successful, the response will include a profile object whose `following` attribute will update appropriately.