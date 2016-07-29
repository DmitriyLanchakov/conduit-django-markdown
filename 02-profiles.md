# Profiles

At the end of the last chapter, we briefly touched on the difference between users and profiles, but I want to dive a little deeper before we start working on profiles.

In software engineering, there is a concept called the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle). The idea is that each class should do one job and it should do that job very well. Why is the Single Responsibility Principle relevant to us? Because it’s the theory behind why we’re separating users and profiles.

Users are for authentication and authorization (permissions). The job of the `User` model is to make sure that a user is allowed to access what they’re trying to access. As an example, a user should be allowed to edit their own email and password. They should not be allowed to change the email and password of another user though.

By contrast, the `Profile` model is all about displaying a user’s information in the UI. Our client will include profile pages for each user. That’s where the name of the `Profile` model comes from. Now we will take some things from the user model because there is an inherit relationship between `Profile`s and `User`s, but we will make it our goal to keep this overlap to a minimum.

Now let’s jump in and create the `Profile` model.

## Creating the Profile model

Create `conduit/apps/profiles/models.py` and add the following:

```python
from django.db import models


class Profile(models.Model):
    # As mentioned, there is an inherent relationship between the Profile and
    # User models. By creating a one-to-one relationship between the two, we
    # are formalizing this relationship. Every user will have one -- and only
    # one -- related Profile model.
    user = models.OneToOneField(
        'authentication.User', on_delete=models.CASCADE
    )

    # Each user profile will have a field where they can tell other users
    # something about themselves. This field will be empty when the user
    # creates their account, so we specify `blank=True`.
    bio = models.TextField(blank=True)

    # In addition to the `bio` field, each user may have a profile image or
    # avatar. Similar to `bio`, this field is not required. It may be blank.
    image = models.URLField(blank=True)

    # A timestamp representing when this object was created.
    created_at = models.DateTimeField(auto_now_add=True)

    # A timestamp reprensenting when this object was last updated.
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.user.username
```

One thing you may notice is that both the `User` and `Profile` models have the `created_at` and `updated_at` fields. These are fields that we will be placing on all of our models, so why don’t we take a few minutes to abstract this into it’s own model?

### Timestamped Model

Create `conduit/apps/core/models.py` and add this snippet:

```python
from django.db import models


class TimestampedModel(models.Model):
    # A timestamp representing when this object was created.
    created_at = models.DateTimeField(auto_now_add=True)

    # A timestamp reprensenting when this object was last updated.
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

        # By default, any model that inherits from `TimestampedModel` should
        # be ordered in reverse-chronological order. We can override this on a
        # per-model basis as needed, but reverse-chronological is a good
        # default ordering for most models.
        ordering = ['-created_at', '-updated_at’]
```

Now go back to `conduit/apps/profiles/models.py` and apply the following changes:

```diff
from django.db import models

+from conduit.apps.core.models import TimestampedModel


-class Profile(models.Model):
+class Profile(TimestampedModel):
    # As mentioned, there is an inherent relationship between the Profile and
    # User models. By creating a one-to-one relationship between the two, we
    # are formalizing this relationship. Every user will have one -- and only
    # one -- related Profile model.
    user = models.OneToOneField(
        'authentication.User', on_delete=models.CASCADE
    )

    # Each user profile will have a field where they can tell other users
    # something about themselves. This field will be empty when the user
    # creates their account, so we specify `blank=True`.
    bio = models.TextField(blank=True)

    # In addition to the `bio` field, each user may have a profile image or
    # avatar. Similar to `bio`, this field is not required. It may be blank.
    image = models.URLField(blank=True)

-    # A timestamp representing when this object was created.
-    created_at = models.DateTimeField(auto_now_add=True)
-
-    # A timestamp reprensenting when this object was last updated.
-    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.user.username
```

Since we want this to apply to the `User` model as well, we’ll need to make a couple of changes there.

Open `conduit/apps/authentication/models.py` and make the following changes:

```diff
import jwt

from datetime import datetime, timedelta

from django.conf import settings
from django.contrib.auth.models import (
    AbstractBaseUser, BaseUserManager, PermissionsMixin
)
from django.db import models

+from conduit.apps.core.models import TimestampedModel

# …

-class User(AbstractBaseUser, PermissionsMixin):
+class User(AbstractBaseUser, PermissionsMixin, TimestampedModel):
    # Each `User` needs a human-readable unique identifier that we can use to
    # represent the `User` in the UI. We want to index this column in the
    # database to improve lookup performance.
    username = models.CharField(db_index=True, max_length=255, unique=True)

    # We also need a way to contact the user and a way for the user to identify
    # themselves when logging in. Since we need an email address for contacting
    # the user anyways, we will also use the email for logging in because it is
    # the most common form of login credential at the time of writing.
    email = models.EmailField(db_index=True, unique=True)

    # When a user no longer wishes to use our platform, they may try to delete
    # there account. That's a problem for us because the data we collect is
    # valuable to us and we don't want to delete it. To solve this problem, we
    # will simply offer users a way to deactivate their account instead of
    # letting them delete it. That way they won't show up on the site anymore,
    # but we can still analyze the data.
    is_active = models.BooleanField(default=True)

    # The `is_staff` flag is expected by Django to determine who can and cannot
    # log into the Django admin site. For most users, this flag will always be
    # falsed.
    is_staff = models.BooleanField(default=False)

-    # A timestamp representing when this object was created.
-    created_at = models.DateTimeField(auto_now_add=True)
-
-    # A timestamp reprensenting when this object was last updated.
-    updated_at = models.DateTimeField(auto_now=True)

    # More fields required by Django when specifying a custom user model.

    # The `USERNAME_FIELD` property tells us which field we will use to log in.
    # In this case, we want that to be the email field.
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

        # …
```

### One-to-One relationships and Django’s Signals framework

In the `Profile` model, we created a one-to-one relationship between `User` and `Profile`. It would be nice if that’s all there was to it and we could call it a day. Unfortunately, that’s not the case. We still have to tell Django that we want to create a `Profile` every time we create a `User`. 

To do this, we will use Django’s [Signals](https://docs.djangoproject.com/en/1.9/topics/signals/) framework. Specifically, we will use the [`post_save`](https://docs.djangoproject.com/en/1.9/ref/signals/#django.db.models.signals.post_save) signal to create the `Profile` instance after the `User` instance.

Start by opening `conduit/apps/authentication/signals.py` and populating the file with the following code:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

from conduit.apps.profiles.models import Profile

from .models import User

@receiver(post_save, sender=User)
def create_related_profile(sender, instance, created, *args, **kwargs):
    # Notice that we're checking for `created` here. We only want to do this
    # the first time the `User` instance is created. If the save that caused
    # this signal to be run was an update action, we know the user already
    # has a profile.
    if instance and created:
        instance.profile = Profile.objects.create(user=instance)
```

This is the signal that will create a profile object, but Django won’t run it by default. Instead, we need to create a custom `AppConfig` class for the `authentication` app and register it with Django.

To do that, open `conduit/apps/authentication/__init__.py` and add the following:

```python
from django.apps import AppConfig


class AuthenticationAppConfig(AppConfig):
    name = 'conduit.apps.authentication'
    label = 'authentication'
    verbose_name = 'Authentication'

    def ready(self):
        import conduit.apps.authentication.signals

# This is how we register our custom app config with Django. Django is smart
# enough to look for the `default_app_config` property of each registered app
# and use the correct app config based on that value.
default_app_config = 'conduit.apps.authentication.AuthenticationAppConfig'
```

Now, when a new user is created, a profile should be created for that user as well. Let’s test this to double check.

### Testing created_related_profile

The first thing we need to do is drop our existing database. None of our existing users will have a profile, so Django will ask us to provide a default value. The problem is that this default will live in a migration and be run for every record we create in the future, which is not what we want.

To drop the database, simply delete the `db.sqlite3` file in the root directory of your project.

After dropping the database, we want to generate new migrations for the `profiles` app. Since this is the first time we’ll be generating migrations for `profiles`, we need to specify that it’s for the `profiles` app:

```bash
~ python manage.py makemigrations profiles
```

NOTE: The `~` in the above snippet should not be typed with the rest of the command. It is simply to identify that we’re running this command from the command line.

After generating the new migrations, run the following to apply the new migrations and create a new database:

```bash
~ python manage.py migrate
```

Now you should be able to use Postman to send a registration request and create a new user with a profile. Go ahead and send that request now.

Now we need to check that the profile was actually created. Run the following from the command line to open a new shell:

```bash
~ python manage.py shell_plus
```

Inside the shell, all we need to do is grab the user we just created and make sure it has a profile:

```bash
>>> u = User.objects.first()
>>> u.profile
<Profile: james>
```

The output you see from `u.profile` will be different based on the username of the user you created. As long as `u.profile` returns a `Profile` instance, we’re all set to move forward.

## Serializing Profile objects

Create `conduit/apps/profiles/serializers.py` with the following content:

```python
from rest_framework import serializers

from .models import Profile


class ProfileSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source='user.username')
    bio = serializers.CharField(allow_blank=True, required=False)
    image = serializers.SerializerMethodField()

    class Meta:
        model = Profile
        fields = ('username', 'bio', 'image',)
        read_only_fields = ('username',)

    def get_image(self, obj):
        if obj.image:
            return obj.image

        return 'https://static.productionready.io/images/smiley-cyrus.jpg'
```

There’s nothing new here. This is very similar to the `UserSerializer` we created in the last chapter. Let’s move on.

## Rendering Profile objects

Since we know we’re going to run into the same issue we had with the user response data not being namespaced under “user”, let’s go ahead and create a `ProfileJSONRenderer`. This renderer will be very similar to the `UserJSONRenderer`, so we’re going to go ahead and just created a `ConduitJSONRenderer` that both `UserJSONRenderer` and `ProfileJSONRenderer` can inherit from. This will let us abstract some parts of the code away so we can avoid duplicating them.

Open up `conduit/apps/core/renderers.py` and add the following:

```python
import json

from rest_framework.renderers import JSONRenderer


class ConduitJSONRenderer(JSONRenderer):
    charset = 'utf-8'
    object_label = 'object'

    def render(self, data, media_type=None, renderer_context=None):
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

The are two difference between `ConduitJSONRenderer` and the `UserJSONRenderer` we created before:

1. In `UserJSONRenderer`, we did not specify an `object_label` property. The reason for this is that we knew what the object label would be for `UserJSONRenderer`: it would be `user`. In this case, however, the object label (or namespace, as referred to before) will change based on what class is inheriting from `ConduitJSONRenderer`. To make this useful, we allow `object_label` to be set dynamically and we default to the value of `object`.
2. `UserJSONRenderer` has to worry about decoding the JWT if it is part of the request. That is a requirement specific to `UserJSONRenderer` that will not be shared by any of renderer. To that end, it doesn’t make sense to include that in `ConduitJSONRenderer`. We will handle updating `UserJSONRenderer` to take care of this case shortly.

Now create `conduit/apps/profiles/renderers.py` and give it the following content:

```python
from conduit.apps.core.renderers import ConduitJSONRenderer


class ProfileJSONRenderer(ConduitJSONRenderer):
    object_label = 'profile'
```

There’s really not anything here before `ProfileJSONRenderer` shares so much functionality with `UserJSONRenderer`. Let’s keep going.

Open `conduit/apps/authentication/renderers.py` and make the following changes:

```diff
-import json
-
-from rest_framework.renderers import JSONRenderer
+from conduit.apps.core.renderers import ConduitJSONRenderer


-class UserJSONRenderer(JSONRenderer):
+class UserJSONRenderer(ConduitJSONRenderer):
-    charset = 'utf-8'
+    object_label = ‘user’

    def render(self, data, media_type=None, renderer_context=None):
-        # If the view throws an error (such as the user can't be authenticated
-        # or something similar), `data` will contain an `errors` key. We want
-        # the default JSONRenderer to handle rendering errors, so we need to
-        # check for this case.
-        errors = data.get('errors', None)
-
        # If we recieve a `token` key as part of the response, it will by a
        # byte object. Byte objects don't serializer well, so we need to
        # decode it before rendering the User object.
        token = data.get('token', None)

-        if errors is not None:
-            # As mentioned about, we will let the default JSONRenderer handle
-            # rendering errors.
-            return super(UserJSONRenderer, self).render(data)

        if token is not None and isinstance(token, bytes):
            # Also as mentioned above, we will decode `token` if it is of type
            # bytes.
            data['token'] = token.decode('utf-8')

-        # Finally, we can render our data under the "user" namespace.
-        return json.dumps({
-            'user': data
-        })
+        return super(UserJSONRenderer, self).render(data)
```

Basically all we’re doing here is removing the parts that are now handled by `ConduitJSONRenderer`.

Everything should still be working exactly as it was for `UserJSONRenderer`. Perform a “Current User” request in Postman to confirm.

## ProfileRetrieveAPIView

Let’s add an endpoint to retrieve information about a specific user.

Create `conduit/apps/profiles/views.py` and add the following:

```python
from rest_framework import status
from rest_framework.generics import RetrieveAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer


class ProfileRetrieveAPIView(RetrieveAPIView):
    permission_classes = (AllowAny,)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def retrieve(self, request, username, *args, **kwargs):
        # Try to retrieve the requested profile and throw an exception if the
        # profile could not be found.
        try:
            # We use the `select_related` method to avoid making unnecessary
            # database calls.
            profile = Profile.objects.select_related('user').get(
                user__username=username
            )
        except Profile.DoesNotExist:
            raise

        serializer = self.serializer_class(profile)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

In the code above, we handle the case where the requested profile doesn’t exist, but we don’t handle it in a very clean way. In particular, we don’t have control over what the error message the client will receive is. Let’s do something about that.

### ProfileDoesNotExist

Create a file called `conduit/apps/profiles/exceptions.py` and add the following:

```python
from rest_framework.exceptions import APIException


class ProfileDoesNotExist(APIException):
    status_code = 400
    default_detail = 'The requested profile does not exist.'
```

This is a very simple exception. In Django REST Framework, any time you want to create a custom exception, you inherit from `APIException`. All you have to do them is specify the `default_detail` and `status_code` properties. The default of this exception can be overriden on a case-by-case basis if you decide that makes the most sense.

Now head over to `conduit/apps/core/exceptions.py` and make the following change to the `core_exception_handler` function:

```diff
def core_exception_handler(exc, context):
    # If an exception is thrown that we don't explicitly handle here, we want
    # to delegate to the default exception handler offered by DRF. If we do
    # handle this exception type, we will still want access to the response
    # generated by DRF, so we get that response up front.
    response = exception_handler(exc, context)
    handlers = {
+        'ProfileDoesNotExist': _handle_generic_error,
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
```

We will handle our custom exception the same way we do a `ValidationError`, but now we have control over the error that the client will see. To bring things full-circle, let’s use `ProfileDoesNotExist` in our view.

Open up `conduit/apps/profiles/views.py` and make this change:

```diff
from rest_framework import status
from rest_framework.generics import RetrieveAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

+from .exceptions import ProfileDoesNotExist
from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer


class ProfileRetrieveAPIView(RetrieveAPIView):
    permission_classes = (AllowAny,)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def retrieve(self, request, username, *args, **kwargs):
        # Try to retrieve the requested profile and throw an exception if the
        # profile could not be found.
        try:
            # We use the `select_related` method to avoid making unnecessary
            # database calls.
            profile = Profile.objects.select_related('user').get(
                user__username=username
            )
        except Profile.DoesNotExist:
-            raise
+            raise ProfileDoesNotExist

        serializer = self.serializer_class(profile)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

Problem solved! Let’s add a url for `ProfileRetrieveAPIView` to our urls file.

Create `conduit/apps/profiles/urls.py` with the following:

```python
from django.conf.urls import url

from .views import ProfileRetrieveAPIView

urlpatterns = [
    url(r'^profiles/(?P<username>\w+)/?$', ProfileRetrieveAPIView.as_view()),
]
```

Like we did with `conduit/apps/authentication/urls.py`, we need to register this new urls file with the main `urlpatterns` variable in `conduit/urls.py`.

Open up `conduit/urls.py` and make this change:

```diff
urlpatterns = [
    url(r'^admin/', admin.site.urls),

    url(r'^api/', include('conduit.apps.authentication.urls', namespace='authentication')),
+    url(r'^api/', include('conduit.apps.profiles.urls', namespace='profiles')),
]
```

## Retrieving a profile with Postman

If you open Postman and look in the “Profiles” folder, there will be a request called “Profile”. Send that request to the server to check that everything we’ve done so far is working. Assuming all is well, we can move on to updating `UserRetrieveUpdateAPIView`.

## Updating UserRetrieveUpdateAPIView

Open up `conduit/apps/authentication/views.py` and make the following changes in the `update` method:

```diff
def update(self, request, *args, **kwargs):
-    serializer_data = request.data.get('user', {})
+    user_data = request.data.get('user', {})
+
+    serializer_data = {
+        ’username': user_data.get('username', request.user.username),
+        ’email': user_data.get('email', request.user.email),
+
+        ’profile': {
+            ’bio': user_data.get('bio', request.user.profile.bio),
+            ’image': user_data.get('image', request.user.profile.image)
+        }
+    }

    # Here is that serialize, validate, save pattern we talked about
    # before.
    serializer = self.serializer_class(
        request.user, data=serializer_data, partial=True
    )
    serializer.is_valid(raise_exception=True)
    serializer.save()

    return Response(serializer.data, status=status.HTTP_200_OK)
```

These changes will let us use the same endpoint for updating the email, password, biography, and image of a user.

We also need to update `UserSerializer` to make the `update` method work with profiles.

## Updating UserSerializer

Open `conduit/apps/authentication/serializers.py` and update the imports like so:

```diff
from django.contrib.auth import authenticate

from rest_framework import serializers

+from conduit.apps.profiles.serializers import ProfileSerializer
+
from .models import User
```

Then we can update `UserSerializer` as follows:

```diff
class UserSerializer(serializers.ModelSerializer):
    """Handles serialization and deserialization of User objects."""

    # Passwords must be at least 8 characters, but no more than 128 
    # characters. These values are the default provided by Django. We could
    # change them, but that would create extra work while introducing no real
    # benefit, so let's just stick with the defaults.
    password = serializers.CharField(
        max_length=128,
        min_length=8,
        write_only=True
    )
+
+    # When a field should be handled as a serializer, we must explicitly say
+    # so. Moreover, `UserSerializer` should never expose profile information,
+    # so we set `write_only=True`.
+    profile = ProfileSerializer(write_only=True)
+
+    # We want to get the `bio` and `image` fields from the related Profile
+    # model.
+    bio = serializers.CharField(source='profile.bio', read_only=True)
+    image = serializers.CharField(source='profile.image', read_only=True)

    class Meta:
        model = User
-        fields = (‘email’, ‘username’, ‘password’, ‘token’,)
+        fields = (
+            'email', 'username', 'password', 'token', 'profile', 'bio',
+            'image',
+        )
    
    # …
```

Finally, we want to change the `update` method on `UserSerializer` to handle profile data. Here are the changes needed to make that work:

```diff
def update(self, instance, validated_data):
    """Performs an update on a User."""

    # Passwords should not be handled with `setattr`, unlike other fields.
    # This is because Django provides a function that handles hashing and
    # salting passwords, which is important for security. What that means
    # here is that we need to remove the password field from the
    # `validated_data` dictionary before iterating over it.
    password = validated_data.pop('password', None)

+    # Like passwords, we have to handle profiles separately. To do that,
+    # we remove the profile data from the `validated_data` dictionary.
+    profile_data = validated_data.pop('profile', {})
+
    for (key, value) in validated_data.items():
        # For the keys remaining in `validated_data`, we will set them on
        # the current `User` instance one at a time.
        setattr(instance, key, value)

    if password is not None:
        # `.set_password()` is the method mentioned above. It handles all
        # of the security stuff that we shouldn't be concerned with.
        instance.set_password(password)

    # Finally, after everything has been updated, we must explicitly save
    # the model. It's worth pointing out that `.set_password()` does not
    # save the model.
    instance.save()

+    for (key, value) in profile_data.items():
+        # We're doing the same thing as above, but this time we're making
+        # changes to the Profile model.
+        setattr(instance.profile, key, value)
+
+    # Save the profile just like we saved the user.
+    instance.profile.save()
+
    return instance
```

With all of these changes made, we’re ready to release our new profile feature to our users! As a sanity check, go back to Postman and make the “Current User” and “Update User” requests in the “Auth” folder to make sure we didn’t break anything during refactoring.

## What’s next?

Next up is the bread and butter of our app: articles. Whether you’re the one reading the articles or you’re the one writing them, they are the most important piece of our app. Without articles, users can’t do anything at all!

In the next chapter, we’ll add a model and serializer for handling articles. We’ll take a look at a new concept of a view set and we’ll add a new signal to our API. See you there!