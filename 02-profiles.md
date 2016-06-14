
### Registering a user with Postman

Now that we’ve created the `User` model and added an endpoint for registering new users, we’re going to run a quick sanity check to make sure we’re on track. To do this, we’re going to use a tool called Postman with a pre-made collection of endpoints.

If you’ve never used Postman before, check out our [Testing Conduit Apps Using Postman](#) guide.

Open up Postman and use the “Register” request inside the “Auth” folder to create a new user.

Awesome! We’re making some real progress now!

There is one thing we need to fix though. Notice how the response from the “Register” request had all of the user’s information at the root level. Our client expects this information to be namespaced under “user”. To do that, we’ll need to create a custom [DRF renderer](http://www.django-rest-framework.org/api-guide/renderers/).

### Rendering User objects

Create a file called `conduit/apps/authentication/renderers.py` and give it the following content:

```python
import json

from rest_framework.renderers import JSONRenderer


class UserJSONRenderer(JSONRenderer):
    charset = 'utf-8'

    def render(self, data, media_type=None, renderer_context=None):
        # If we recieve a `token` key as part of the response, it will by a
        # byte object. Byte objects don't serializer well, so we need to
        # decode it before rendering the User object.
        token = data.get('token', None)

        if token is not None and isinstance(token, bytes):
            # Also as mentioned above, we will decode `token` if it is of type
            # bytes.
            data['token'] = token.decode('utf-8')

        # Finally, we can render our data under the "user" namespace.
        return json.dumps({
            'user': data
        })

```

There’s nothing new or interesting happening here, so just read through the comments in the snippet and then we can move on.

Now open `conduit/apps/authentication/views.py` and import `UserJSONRenderer` by adding the following line to the top of your file:

```python
from .renderers import UserJSONRenderer
```

You’ll also need to set the `renderer_classes` property of the `RegistrationAPIView` class like so:

```diff
class UserRetrieveUpdateAPIView(RetrieveUpdateAPIView):
    permission_classes = (IsAuthenticated,)
+    renderer_classes = (UserJSONRenderer,)
    serializer_class = UserSerializer

    # … Other things
```

With `UserJSONRenderer` in place, go ahead and use the “Register” Postman request to create a new user. Notice how, this time, the response is inside the “user” namespace.

## Logging users in

Since users can now register for Conduit, we need to build a way for them to log in to their account. In this lesson we will add the serializer and view needed for users to log in. We will also start looking at how our API should handle errors.

### LoginSerializer

Open `conduit/apps/authentication/serializers.py` and add the following import to the top of the file:

```python
from django.contrib.auth import authenticate
```

After that, create the following serializer in the same file:

```python
class LoginSerializer(serializers.Serializer):
    email = serializers.CharField(max_length=255)
    username = serializers.CharField(max_length=255, read_only=True)
    password = serializers.CharField(max_length=128, write_only=True)
    token = serializers.CharField(max_length=255, read_only=True)

    def validate(self, data):
        # The `validate` method is where we make sure that the current
        # instance of `LoginSerializer` has "valid". In the case of logging a
        # user in, this means validating that they've provided an email
        # and password and that this combination matches one of the users in
        # our database.
        email = data.get('email', None)
        password = data.get('password', None)

        # As mentioned above, an email is required. Raise an exception if an
        # email is not provided.
        if email is None:
            raise serializers.ValidationError(
                'An email address is required to log in.'
            )

        # As mentioned above, a password is required. Raise an exception if a
        # password is not provided.
        if password is None:
            raise serializers.ValidationError(
                'A password is required to log in.'
            )

        # The `authenticate` method is provided by Django and handles checking
        # for a user that matches this email/password combination. Notice how
        # we pass `email` as the `username` value. Remember that, in our User
        # model, we set `USERNAME_FIELD` as `email`.
        user = authenticate(username=email, password=password)

        # If no user was found matching this email/password combination then
        # `authenticate` will return `None`. Raise an exception in this case.
        if user is None:
            raise serializers.ValidationError(
                'A user with this email and password was not found.'
            )

        # Django provides a flag on our `User` model called `is_active`. The
        # purpose of this flag to tell us whether the user has been banned
        # or otherwise deactivated. This will almost never be the case, but
        # it is worth checking for. Raise an exception in this case.
        if not user.is_active:
            raise serializers.ValidationError(
                'This user has been deactivated.'
            )

        # The `validate` method should return a dictionary of validated data.
        # This is the data that is passed to the `create` and `update` methods
        # that we will see later on.
        return {
            'email': user.email,
            'username': user.username,
            'token': user.token
        }
```

With the serializer in place, let’s move on to creating the view.

### LoginAPIView

Open `conduit/apps/authentication/views.py` and update the following import:

```diff
-from .serializers import RegistrationSerializer
+from .serializers import (
+    LoginSerializer, RegistrationSerializer
+)
```

Then add the new login view:

```python
class LoginAPIView(APIView):
    permission_classes = (AllowAny,)
    renderer_classes = (UserJSONRenderer,)
    serializer_class = LoginSerializer

    def post(self, request):
        user = request.data.get('user', {})

        # Notice here that we do not call `serializer.save()` like we did for
        # the registration endpoint. This is because we don't actually have
        # anything to save. Instead, the `validate` method on our serializer
        # handles everything we need.
        serializer = self.serializer_class(data=user)
        serializer.is_valid(raise_exception=True)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

Open `conduit/apps/authentication/urls.py` and update the following import:

```diff
-from .views import RegistrationAPIView
+from .views import LoginAPIView, RegistrationAPIView
```

And add a new rule to the `urlpatterns` list:

```diff
urlpatterns = [
    url(r'^users/?$', RegistrationAPIView.as_view()),
+    url(r'^users/login/?$', LoginAPIView.as_view()),
]
```

### Logging a user in with Postman

At this point, a user should be able to log in by hitting the new login endpoint. Let’s test this out. Open up Postman and use the “Login” request to log in with one of the users you created previously. If the login attempt was successful, the response will include a token that can be used in the future when making requests that require the user be authenticated.

There is something else we need to handle here, though. Specifically, try using the “Login” request to log in with an invalid email/password combination. Notice the error response. There are two problems with this.

First of all, `non_field_errors` sounds strange. Usually this key coresponds to whatever field caused the serializer to fail validation. In our case, since we overrode the `validate` method instead of a field-specific method such as `validate_email`, Django REST Framework didn’t know what field to attribute the error to. In this case, there is a default that can be used. By default, the default is `non_field_errors`. Since our client will be using this key to display errors, we’re going to change this to simply say `error`.

Secondly, the client expects any errors to be namespaced under the `errors` key in a JSON response, similar to how we namespaced the login and register requests under the `user` key. We will accomplish this by overriding Django REST Framework’s default error handling.

### Overriding EXCEPTION_HANDLER and NON_FIELD_ERRORS_KEY

One of DRF’s settings is called `EXCEPTION_HANDLER`. The default exception handler simply returns a dictionary of errors. We want our errors namespaced under the `errors` key, so we’re going to have to override `EXCEPTION_HANDLER`. We will also override `NON_FIELD_ERRORS_KEY` as mentioned earlier.

Let’s start by creating `conduit/apps/core/exceptions.py` and adding the following snippet:

```python
from rest_framework.views import exception_handler

def core_exception_handler(exc, context):
    # If an exception is thrown that we don't explicitly handle here, we want
    # to delegate to the default exception handler offered by DRF. If we do
    # handle this exception type, we will still want access to the response
    # generated by DRF, so we get that response up front.
    response = exception_handler(exc, context)
    handlers = {
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

def _handle_generic_error(exc, context, response):
    # This is about the most straightforward exception handler we can create.
    # We take the response generated by DRF and wrap it in the `errors` key.
    response.data = {
        'errors': response.data
    }

    return response
```

Will that taken care of, open up `conduit/settings.py` and add a new setting to the bottom of the file called `REST_FRAMEWORK`, like so:

```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'conduit.apps.core.exceptions.core_exception_handler',
    'NON_FIELD_ERRORS_KEY': 'error',
}
```

This is how we override settings in DRF. We will add one more setting in a bit when we start writing views that require the user to be authenticated.

Let’s try sending another login request using Postman. Be sure to use an email/password combination that is invalid.

### Updating UserJSONRenderer

Uh oh! Still not quite what we want. We’ve got the `errors` key now, but everything is namespaced under the `user` key. That’s not good.

Let’s update `UserJSONRenderer` to check for the `errors` key and do things a bit differently. Open up `conduit/apps/authentication/renderers.py` and make these changes:

```diff
class UserJSONRenderer(JSONRenderer):
    charset = 'utf-8'

    def render(self, data, media_type=None, renderer_context=None):
+        # If the view throws an error (such as the user can't be authenticated
+        # or something similar), `data` will contain an `errors` key. We want
+        # the default JSONRenderer to handle rendering errors, so we need to
+        # check for this case.
+        errors = data.get('errors', None)

        # If we recieve a `token` key as part of the response, it will by a
        # byte object. Byte objects don't serializer well, so we need to
        # decode it before rendering the User object.
        token = data.get('token', None)

+        if errors is not None:
+            # As mentioned about, we will let the default JSONRenderer handle
+            # rendering errors.
+            return super(UserJSONRenderer, self).render(data)

        if token is not None and isinstance(token, bytes):
            # Also as mentioned above, we will decode `token` if it is of type
            # bytes.
            data['token'] = token.decode('utf-8')

        # Finally, we can render our data under the "user" namespace.
        return json.dumps({
            'user': data
        })
```

Now send the login request with Postman one more time and all should be well.

## Retrieving and updating users

Users can register new accounts and log in to those accounts. What’s next? Users will need a way to retrieve their information and update that information. Let’s get going on that before we move on to creating user profiles.

### UserSerializer

We’re going to create one more serializer for profile. We’ve got serializers for login and register requests, but we need to be able to serializer user objects too.

Open `conduit/apps/authentication/serializers.py` and add the following:

```python
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

    class Meta:
        model = User
        fields = ('email', 'username', 'password', 'token',)

        # The `read_only_fields` option is an alternative for explicitly
        # specifying the field with `read_only=True` like we did for password
        # above. The reason we want to use `read_only_fields` here is because
        # we don't need to specify anything else about the field. For the
        # password field, we needed to specify the `min_length` and 
        # `max_length` properties too, but that isn't the case for the token
        # field.
        read_only_fields = ('token',)


    def update(self, instance, validated_data):
        """Performs an update on a User."""

        # Passwords should not be handled with `setattr`, unlike other fields.
        # This is because Django provides a function that handles hashing and
        # salting passwords, which is important for security. What that means
        # here is that we need to remove the password field from the
        # `validated_data` dictionary before iterating over it.
        password = validated_data.pop('password', None)

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

        return instance
```

One thing worth pointing out about the above serializer is that we do not explicitly define the `create` method. DRF provides a default `create` method for all instances of `serializers.ModelSerializer`, so it’s still possible to create a user with this serializer, but we don’t want to do that. Creation of a `User` should be handled by `RegistrationSerializer`.

### UserRetrieveUpdateAPIView

Open up `conduit/apps/authentication/views.py` and update the imports like so:

```diff
from rest_framework import status
+from rest_framework.generics import RetrieveUpdateAPIView
-from rest_framework.permissions import AllowAny
+from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

from .renderers import UserJSONRenderer
from .serializers import (
-   LoginSerializer, RegistraitonSerializer
+    LoginSerializer, RegistrationSerializer, UserSerializer,
)
```

Below the imports, create a new view called `UserRetrieveUpdateAPIView`:

```python
class UserRetrieveUpdateAPIView(RetrieveUpdateAPIView):
    permission_classes = (IsAuthenticated,)
    renderer_classes = (UserJSONRenderer,)
    serializer_class = UserSerializer

    def retrieve(self, request, *args, **kwargs):
        # There is nothing to validate or save here. Instead, we just want the
        # serializer to handle turning our `User` object into something that
        # can be JSONified and sent to the client.
        serializer = self.serializer_class(request.user)

        return Response(serializer.data, status=status.HTTP_200_OK)

    def update(self, request, *args, **kwargs):
        serializer_data = request.data.get('user', {})

        # Here is that serialize, validate, save pattern we talked about
        # before.
        serializer = self.serializer_class(
            request.user, data=serializer_data, partial=True
        )
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_200_OK)
```

Now go over to `apps/conduit/authentication/urls.py` and update the imports to include `UserRetrieveUpdateAPIView`:

```diff
-from .views import LoginAPIView, RegistrationAPIView
+from .views import (
+    LoginAPIView, RegistrationAPIView, UserRetrieveUpdateAPIView
+)
```

And add a new route to `urlpatterns`:

```diff
urlpatterns = [
+    url(r'^user/?$', UserRetrieveUpdateAPIView.as_view()),
    url(r'^users/?$', RegistrationAPIView.as_view()),
    url(r'^users/login/?$', LoginAPIView.as_view()),
]
```

Open up Postman again and send the “Current User” request. You should get an error with a response that looks like this:

```json
{
  "user": {
    "detail": "Authentication credentials were not provided."
  }
}
```

## Authenticating Users

Django has this idea of authentication backends. Without going into too much detail, a backend is essentially a plan for deciding whether a user is authenticated. Because neither Django nor Django REST Framework support JWT authentication out-of-the-box, we’ll need to create a custom backend.

Create and open ‘conduit/apps/authentication/backends.py` and add the following code:

```python
import jwt

from django.conf import settings

from rest_framework import authentication, exceptions

from .models import User


class JWTAuthentication(authentication.BaseAuthentication):
    authentication_header_prefix = 'Token'

    def authenticate(self, request):
        """
        The `authenticate` method is called on every request, regardless of
        whether the endpoint requires authentication. 

        `authenticate` has two possible return values:

        1) `None` - We return `None` if we do not wish to authenticate. Usually
                    this means we know authentication will fail. An example of
                    this is when the request does not include a token in the
                    headers.

        2) `(user, token)` - We return a user/token combination when 
                             authentication was successful.

        If neither of these two cases were met, that means there was an error.
        In the event of an error, we do not return anything. We simple raise
        the `AuthenticationFailed` exception and let Django REST Framework
        handle the rest.
        """
        request.user = None

        # `auth_header` should be an array with two elements: 1) the name of
        # the authentication header (in this case, "Token") and 2) the JWT 
        # that we should authenticate against.
        auth_header = authentication.get_authorization_header(request).split()
        auth_header_prefix = self.authentication_header_prefix.lower()

        if not auth_header:
            return None

        if len(auth_header) == 1:
            # Invalid token header. No credentials provided. Do not attempt to
            # authenticate.
            return None

        elif len(auth_header) > 2:
            # Invalid token header. Token string should not contain spaces. Do
            # not attempt to authenticate.
            return None

        # The JWT library we're using can't handle the `byte` type, which is
        # commonly used by standard libraries in Python 3. To get around this,
        # we simply have to decode `prefix` and `token`. This does not make for
        # clean code, but it is a good decision because we would get an error
        # if we didn't decode these values.
        prefix = auth_header[0].decode('utf-8')
        token = auth_header[1].decode('utf-8')

        if prefix.lower() != auth_header_prefix:
            # The auth header prefix is not what we expected. Do not attempt to
            # authenticate.
            return None

        # By now, we are sure there is a *chance* that authentication will
        # succeed. We delegate the actual credentials authentication to the
        # method below.
        return self._authenticate_credentials(request, token)

    def _authenticate_credentials(self, request, token):
        """
        Try to authenticate the given credentials. If authentication is
        successful, return the user and token. If not, throw an error.
        """
        try:
            payload = jwt.decode(token, settings.SECRET_KEY)
        except:
            msg = 'Invalid authentication. Could not decode token.'
            raise exceptions.AuthenticationFailed(msg)

        try:
            user = User.objects.get(pk=payload['id'])
        except User.DoesNotExist:
            msg = 'No user matching this token was found.'
            raise exceptions.AuthenticationFailed(msg)

        if not user.is_active:
            msg = 'This user has been deactivated.'
            raise exceptions.AuthenticationFailed(msg)

        return (user, token)
```

{x: create authentication backends}
Create `conduit/apps/authentication/backends.py`

There is a lot of logic and lot of exceptions being thrown in this file, but the code is pretty straight forward. All we’ve done is list conditions where the user would not be authenticated and throw an exception if any of those conditions are true.

There isn’t any extra reading to do here, but feel free to check out the docs for the [PyJWT](https://pyjwt.readthedocs.io/en/latest/) library, if you’re interested.

### Telling DRF about our authentication backend

We must explicitly tell Django REST Framework which authentication backend we want to use, similar to how we told Django to use our custom User model.

Open up `conduit/settings.py` and add the following snippet to the end of the file:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'authentication.backends.JWTAuthentication',
    ),
}
```

{x: register our jwt authentication backend} 
Register our JWT authentication backend with Django REST Framework.

## Retrieving and updating users with Postman

With our new authentication backend in place, the authentication error we saw a while ago should be gone. Test this by opening Postman and sending another “Current User” request. The request should be successful and you should see the information about your user in the response.

Remember that we created an update endpoint at the same time we created the retrieve endpoint. Let’s test this one too. Send the request labeled “Update User” in the “Auth” folder in Postman. If you used the default body, the email of your user should have changed. Feel free to play around with these requests to make changes to your user.

## On to better things

That’s all there is for this chapter. We’ve created a user model and serialized users in three different ways. There are four shiny new endpoints that let users register, login, and retrieve and update their information. We’re off to a really good start here!

Next up, we’re going to create profiles for our users. You may have noticed that the `User` model is pretty bare-bones. We only included things essential for authentication. Other information such as a biography and avatar URL will go in the `Profile` model that we’ll work with in the next chapter.