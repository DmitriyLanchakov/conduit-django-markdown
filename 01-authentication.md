# Authentication

Users have to log in before they can post an article. That means they need to have an account. This chapter is about building the login and registration systems. This will be, by far, the longest chapter in this course. I recommend taking as many breaks as you need. We have tried to break the course up into sections. The end of each section is a good place to take a break if you need to.

## Authentication in Django

Django comes with a session-based authentication system that works out of the box. It includes all of the models, views, and templates you need to let users log in and create a new account. Here’s the rub though: Django’s authentication only works with the traditional HTML request-response cycle.

What do we mean by “the traditional HTML request-response cycle”? Historically, when a user wanted to perform some action (such as creating a new account), the user would fill out a form in their web browser. When they clicked the “Submit” button, the browser would make a request — which included the data the user had typed into the registration form — to the server, the server would process that request, and it would respond with HTML or redirect the browser to a new page. This is what we mean when we talk about doing a “full page refresh”.

Why is knowing that Django’s built-in authentication only works with the traditional HTML request-response cycle important? Because the client we’re building this API for does not adhere to this cycle. Instead, the client expects the server to return JSON instead of HTML. By returning JSON, we can let the client decide what it should do next instead of letting the server decide. With a JSON request-response cycle, the server receives data, processes it, and returns a response (just like in the HTML request-response cycle), but the response does not actually control the browser’s behavior. It just tells us what the result of the request was.

Luckily, the team behind Django realized that the trend of web development was moving in this direction. They also knew that some projects may not want to use the built-in models, views, and templates. They may choose to use their own custom versions instead. To make sure all of the effort that went into building Django’s built-in authentication system wasn’t wasted, they decided to make it possible to use the most important parts while maintaining the ability to customize the end result.

We will dive into this more later in the chapter. For now, here’s what you want to know:

1. We will be creating our own `User` model to replace Django’s.
2. We will have to write our own views to support returning JSON instead of HTML.
3. Because we won’t be using HTML, we have no need for Django’s built-in login and register templates.

If you’re wondering what’s let for us to use, that’s a fair question. This goes back to what we talked about earlier about Django making it possible to use the core parts of authentication without using the default authentication system.

{x: read using the django authentication system} 
Read [Using the Django authentication system](https://docs.djangoproject.com/es/1.9/topics/auth/default/) to learn more about how Django’s default authentication works.

{x: read customizing authentication in django} 
Read [Customizing authentication in Django](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/) to get a better idea about what lies ahead in the rest of this chapter. We will return to this page later.

## Session-based authentication

By default, Django uses sessions for authentication. We alluded to this earlier. Before going further, we should talk about what this means, why it’s important, what token-based authentication and JSON Web Tokens (JWTs, for short) are, and which one we’ll be using in this course.

In Django, sessions are stored as cookies. These sessions, along with some built-in middleware and request objects, ensure that there is a user available on every request. The user can be accessed as `request.user`. When the user is logged in, `request.user` is an instance of the `User` class. When they’re logged out, `request.user` is an instance of the `AnonymousUser` class. Whether the user is authenticated or not, `request.user` will always exist.

What’s the difference? Put simply, anytime you want to know if the current user is authentication, you can use `request.user.is_authenticated()` which will return `True` if the user is authenticated and `False` if they aren’t. If `request.user` is an instance of `AnonymousUser`, `request.user.is_authenticated()` will always return `False`. This allows the developer (you!) to turn `if request.user is not None and request.user.is_authenticated():` into `if request.user.is_authenticated():`. Less typing is a good thing in this case!

In our case, the client and the server will be running at different locations. The server will be running at `http://localhost:3000/` and the client will be at `http://localhost:5000/`. The browser considers these two locations to be on different domains, similar to running the server on `http://www.server.com` and running the client on `http://www.client.com`. We will not be allowing external domains access to our cookies, so we have to find another alternative solution to using sessions.

If you’re wondering why we won’t be allowing access to our cookies, you should check out the articles on Cross-Origin Resource Sharing (CORS) and Cross Site Request Forgery (CSRF) linked below. If you just want to start coding, check the boxes and move on.

{x: read http access control} 
Read [HTTP access control (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) to learn more about Cross-Origin Resource Sharing (CORS).

{x: read cross site request forgery protection} 
Read [Cross Site Request Forgery protection](https://docs.djangoproject.com/ja/1.9/ref/csrf/) to learn more about how Django protects against CSRF attacks.

## Token-based authentication

The most common alternative to session-based authentication is token-based authentication, and we will be using a specific form of token-based authentication to secure our application.

With token-based auth, the server provides the client with a token when a successful login request is made. This token is unique to the user logging in and it’s stored in the database along with the user’s ID. The client is expected to send the token with future requests so the server can identify the user. The server does this by searching the database table containing all of the tokens that have been created. If a matching token is found, the server goes on to verify that the token is still valid. If no matching token is found, then we say the user is not authenticated.

Because tokens are stored in the database and not in cookies, token-based authentication will suit our needs.

### Verifying tokens

We always have the option of storing more than just the user’s ID with their token. We can also store things such as a date on which the token will expire. In this example, to verify the token, we would need to make sure that this expiration does has not passed. If it has, then the token is not valid, so we delete it from the database and ask the user to log in again.

## JSON Web Tokens

JSON Web Token (JWT, for short) is an open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) that defines a compact and self-contained way for securely transmitting information between two parties. You can think of JWTs as authentication tokens on steroids.

Remember when I said we’ll be using a specific form of token-based authentication? JWTs are what I was referring to.

{x: read introduction to json web tokens} 
Read [Introduction to JSON Web Tokens](https://tools.ietf.org/html/rfc7519) to learn more about what JWTs are and how they work.

### Why are JSON Web Tokens better than regular tokens?

There are a few benefits we get when going with JWTs over regular tokens:

1. JWT is an open standard. That means that all implementations of JWT should be fairly similar, which is a benefit when working with different languages and technologies. Regular tokens are more free-form and it’s left up to the application developer to decide how best to implement the tokens.
2. JWTs can contain all of the information about the user, which is a convenient for the client.
3. There are libraries that can handle the heavy lifting here. Rolling your own authentication is dangerous, so we leave the important stuff to battle-tested libraries that we can trust.

## Creating the User model

How about we get started?

The models we will use for authentication will be stored in a file called `conduit/apps/authentication/models.py`. If you cloned the repository earlier in the course, you’ll notice that the directory `conduit/apps/authentication/` already exists. However, the file `models.py` does not. You’ll want to create this file yourself.

{x: create authentication/models.py} 
Create `conduit/apps/authentication/models.py`.

{x: read substituting a custom user model} 
Read [Substituting a custom User model](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#substituting-a-custom-user-model).

We need to create our custom `User` model and a `Manager` class to go with it.

{x: read up on manager classes} Read [Managers](https://docs.djangoproject.com/en/1.9/topics/db/managers/) to learn more about what a `Manager` class is.

Here is the code for `authentication/models.py`:

```python
import jwt

from datetime import datetime, timedelta

from django.conf import settings
from django.contrib.auth.models import (
    AbstractBaseUser, BaseUserManager, PermissionsMixin
)
from django.db import models


class UserManager(BaseUserManager):
    """
    Django requires that custom users define their own Manager class. By
    inheriting from `BaseUserManager`, we get a lot of the same code used by
    Django to create a `User` for free. 

    All we have to do is override the `create_user` function which we will use
    to create `User` objects.
    """

    def create_user(self, username, email, password=None):
        """Create and return a `User` with an email, username and password."""
        if username is None:
            raise TypeError('Users must have a username.')

        if email is None:
            raise TypeError('Users must have an email address.')

        user = self.model(username=username, email=self.normalize_email(email))
        user.set_password(password)
        user.save()

        return user

    def create_superuser(self, username, email, password):
        """
        Create and return a `User` with superuser powers.

        Superuser powers means that this use is an admin that can do anything
        they want.
        """
        if password is None:
            raise TypeError('Superusers must have a password.')

        user = self.create_user(username, email, password)
        user.is_superuser = True
        user.is_staff = True
        user.save()

        return user


class User(AbstractBaseUser, PermissionsMixin):
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

    # A timestamp representing when this object was created.
    created_at = models.DateTimeField(auto_now_add=True)

    # A timestamp reprensenting when this object was last updated.
    updated_at = models.DateTimeField(auto_now=True)

    # More fields required by Django when specifying a custom user model.

    # The `USERNAME_FIELD` property tells us which field we will use to log in.
    # In this case, we want that to be the email field.
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    # Tells Django that the UserManager class defined above should manage
    # objects of this type.
    objects = UserManager()

    def __str__(self):
        """
        Returns a string representation of this `User`.

        This string is used when a `User` is printed in the console.
        """
        return self.email

    @property
    def token(self):
        """
        Allows us to get a user's token by calling `user.token` instead of
        `user.generate_jwt_token().

        The `@property` decorator above makes this possible. `token` is called
        a "dynamic property".
        """
        return self._generate_jwt_token()

    def get_full_name(self):
        """
        This method is required by Django for things like handling emails.
        Typically, this would be the user's first and last name. Since we do
        not store the user's real name, we return their username instead.
        """
        return self.username

    def get_short_name(self):
        """
        This method is required by Django for things like handling emails.
        Typically, this would be the user's first name. Since we do not store
        the user's real name, we return their username instead.
        """
        return self.username

    def _generate_jwt_token(self):
        """
        Generates a JSON Web Token that stores this user's ID and has an expiry
        date set to 60 days into the future.
        """
        dt = datetime.now() + timedelta(days=60)

        token = jwt.encode({
            'id': self.pk,
            'exp': int(dt.strftime('%s'))
        }, settings.SECRET_KEY, algorithm='HS256')

        return token.decode('utf-8')
```

{x: create the user model}
Create the `User` model.

{x: create the usermanager class}
Create the `UserManager` class.

Our explanations of the code provided throughout this course will be intentionally short. Between the comments in the code and the resources we will link to, we’re confident that you can find the information you need. 

Glance over the above snippet for a few minutes and then do the following:

{x: read customuser} 
Read up on what a [custom `User`](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#django.contrib.auth.models.CustomUser) model is supposed to do.

{x: read about abstractbaseuser and permissionsmixin} 
Read up on [`AbstractBaseUser`](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#django.contrib.auth.models.AbstractBaseUser) and [`PermissionsMixin`](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#django.contrib.auth.models.PermissionsMixin), which are two classes that our custom `User` model inherits from.

{x: read about field options} 
Check out the docs on Django [Field options](https://docs.djangoproject.com/en/1.9/ref/models/fields/) to learn more about options like `db_index` and `unique`.

{x: read baseusermanager} 
Read up on what [`BaseUserManager`](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#django.contrib.auth.models.BaseUserManager) lets us do.

{x: read customerusermanager} 
Read up on what a [custom `UserManager`](https://docs.djangoproject.com/es/1.9/topics/auth/customizing/#django.contrib.auth.models.CustomUserManager) class is supposed to do.

The next thing you’ll want to do is to tell Django about our custom `User` model. Do this by opening `conduit/settings.py` and add the following to the bottom of the file:

```python
# Tell Django about the custom `User` model we created. The string
# `authentication.User` tells Django we are referring to the `User` model in
# the `authentication` module. This module is registered above in a setting
# called `INSTALLED_APPS`.
AUTH_USER_MODEL = 'authentication.User'
```

{x: install custom user model} 
Tell Django about our custom `User` model.

{x: read about auth_user_model} 
Read up on [`AUTH_USER_MODEL`](https://docs.djangoproject.com/es/1.9/ref/settings/#std:setting-AUTH_USER_MODEL) and what it is used for.

### Creating and running migrations

Migrations are files that tell the database what should be changed. In this case, the migrations we’re about to generate will tell Django that we’ve created a new model called `User`.

Before we do anything, if you have run `python manage.py makemigrations` or `python manage.py migrate` already, please delete the `db.sqlite3` file from the root directory of your project. Django gets unhappy if you change `AUTH_USER_MODEL` after creating the database. The easiest way around this is to just drop that database and start anew.

Now you’re ready to create and apply any migrations. After that we can create our first user.

To create migrations, you’ll want to run the following in your console:

```
python manage.py makemigrations
```

This command will create the default migrations for our new Django project. However, it will not create migrations for new apps inside of our project. The first time we want to create migrations for a new app, we have to be more explicit about it.

To create a set of migrations for the `authentication` app, run the following:

```
python manage.py makemigrations authentication
```

This will create the initial migration for the `authentication` app. In the future, whenever you want to generate migrations for the `authentication` app, you only need to run `python manage.py makemigrations`.

{x: make default migrations}
Make the default migrations by running `python manage.py makemigrations`.

{x: make authentication migrations}
Make the initial migrations for the `authentication` app by running `python manage.py makemigrations authentication`.

With the migrations created, we can apply them by running this command:

```
python manage.py migrate
```

Unlike the `makemigrations` command, you never need to specify the app to be migrated when running the `migrate` command.

{x: apply new migrations authentication}
Apply the newly create migrations by running `make migrate`.

### Our first user

We’ve create our `User` model and our database is up and running. The next thing to do is to create our first `User` object. Since this is the user we will be using to test the site, it’s probably beneficial to make it a superuser. That way we have access to everything we need.

Create your first user by running the following command:

```
python manage.py createsuperuser
```

Django will ask you a few questions such as your email address, your username, and your password. Once you answer all of these questions, your new user will be created. Congratulations!

{x: create first user}
Create your first user by running `python manage.py createsuperuser`.

To test that the user has been created, let’s start by opening a Django shell from the command line:

```
python manage.py shell_plus
```

If you’ve used Django before, you may be familiar with the `shell` command, but not with `shell_plus`. `shell_plus` is provided by a library called `django-extensions`, which has been included in the boilerplate project you cloned before starting this course. Why is `shell_plus` useful? Because it automatically imports the models for each app in the `INSTALLED_APPS` setting. It can also be set up to automatically import other utilities, but we won’t need to do that for this project.

Once the shell is open, run the following:

NOTE: Only type what appears on lines preceded by `>>>`. Lines that are not preceded by `>>>` are output from the previous command.

```shell
>>> user = User.objects.first()
>>> user.username
‘james’
>>> user.token
'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE0Njk0MDY2OTksImlkIjoxfQ.qSnwWVD4PJKhKxgLxY0H5mkTE51QnMWv_kqNJVau1go'
```

If all went well, you should see output similar to the above.

## Registering new users

At the moment a user can’t do anything interesting. Our next task is to create an endpoint for registering new users.

### RegistrationSerializer

Start by creating `conduit/apps/authentication/serializers.py` and typing the following code:

```python
from rest_framework import serializers

from .models import User


class RegistrationSerializer(serializers.ModelSerializer):
    """Serializers registration requests and creates a new user."""

    # Ensure passwords are at least 8 characters long, no longer than 128
    # characters, and can not be read by the client.
    password = serializers.CharField(
        max_length=128,
        min_length=8,
        write_only=True
    )

    # The client should not be able to send a token along with a registration
    # request. Making `token` read-only handles that for us.
    token = serializers.CharField(max_length=255, read_only=True)

    class Meta:
        model = User
        # List all of the fields that could possibly be included in a request
        # or response, including fields specified explicitly above.
        fields = ['email', 'username', 'password', 'token']

    def create(self, validated_data):
        # Use the `create_user` method we wrote earlier to create a new user.
        return User.objects.create_user(**validated_data)
```

{x: create authentication serializers file} 
Create `conduit/apps/authentication/serializers.py`.

Read through the code, paying special attention to the comments, and then move on when you’re done.

{x: read about modelserializer}
Read about [`ModelSerializer`](http://www.django-rest-framework.org/api-guide/serializers/#modelserializer).

In the code above, we create a class `RegistrationSerializer` that inherits from `serializers.ModelSerializer`. `serializers.ModelSerializer` is just an abstraction on top of `serializers.Serializer`, which you probably remember from the Django REST Framework tutorial. `ModelSerializer` handles a few things relevant to serializing Django models for us, so we don’t have to.

Another thing to point out about `ModelSerializer` is that it allows you to specify two methods: `create` and `update`. In the above example we wrote our own `create` method using `User.objects.create_user`. We did not specify the `update` method though. In this case, DRF will use it’s own default `update` method to update a user.

### RegistrationAPIView

We can now serialize requests and responses for registering a user. Next, we want to create a view to use as an endpoint. That way, the client will have a URL to hit to create a new user.

Create `conduit/apps/authentication/views.py` and type the following:

```python
from rest_framework import status
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from rest_framework.views import APIView

from .serializers import RegistrationSerializer


class RegistrationAPIView(APIView):
    # Allow any user (authenticated or not) to hit this endpoint.
    permission_classes = (AllowAny,)
    serializer_class = RegistrationSerializer

    def post(self, request):
        user = request.data.get('user', {})

        # The create serializer, validate serializer, save serializer pattern
        # below is common and you will see it a lot throughout this course and
        # your own work later on. Get familiar with it.
        serializer = self.serializer_class(data=user)
        serializer.is_valid(raise_exception=True)
        serializer.save()

        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

Let’s talk about a couple new things in this snippet:

1. The `permission_classes` property is how we decide who can use this endpoint. We can restrict this to authenticated users or admin users. We can also allow users who are authenticated or not based on whether this endpoint they’re hitting is “safe” — that means the endpoint is a `GET`, `HEAD`, or `OPTIONS` request. For the purposes of this course, you only need to know about `GET` requests. We will talk more about `permissions_classes` later on.
2. The create serializer, validate serializer, save serializer pattern you see inside the `post` method is very common when using DRF. You’ll want to familiarize yourself with this pattern as you will be using it a lot.

{x: create authentication views file} 
Create `conduit/apps/authentication/views.py`.

{x: read up on drf permissions}
Read up on Django Rest Framework's [Permissions](http://www.django-rest-framework.org/api-guide/permissions/).

That’s the only view we’re going to write for now. Let’s expose `RegistrationAPIView` to our clients by adding it to the list of URLs.

Make a file called `conduit/apps/authentication/urls.py` with the following content:

```python
from django.conf.urls import url

from .views import RegistrationAPIView

urlpatterns = [
    url(r'^users/?$', RegistrationAPIView.as_view()),
]
```

{x: create authentication urls file} 
Create `conduit/apps/authentication/urls.py`.

If you’re coming from another framework like Rails, it may seem strange that the first argument we pass to the `url` method is a regular expression. It takes some getting use to, but this turns out to be very powerful. For instance, the route about will match both `/users` and `/users/`. The reason for this is that the Django convention is to suffix routes with a trailing slash. Unfortunately, that is at odds with our spec, which prefers to not use trailing slashes. Luckily, the team behind Django has us covered.

Another difference between Django and Rails is that, in Rails, it is common to put all of your routes in a single file. While you can do this in Django, it is considered best practice to break your routes into smaller files for organization reasons. That’s what we’ve done here. Now let’s include the above file in our global URLs file.

Open `conduit/urls.py` and you’ll see the following line near the top of the file:

```python
from django.conf.urls import url
```

The first thing we want to do is import a method called `include` from `django.conf.urls`:

```diff
-from django.conf.urls import url
+from django.conf.urls import include, url
```

The `include` method let’s us include another `urls.py` file without having to do a bunch of work like importing and then re-registering the routes in this file.

Further down, you’ll see the following:

```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),
]
```

Let’s update this to include our new `urls.py` file:

```diff
urlpatterns = [
    url(r'^admin/', admin.site.urls),
+
+    url(r'^api/', include('conduit.apps.authentication.urls', namespace='authentication')),
]
```

{x: add authentication urls to base urls} 
Include the URLs from the `authentication` app in the main `urls.py` file.

### Registering a user with Postman

Now that we’ve created the `User` model and added an endpoint for registering new users, we’re going to run a quick sanity check to make sure we’re on track. To do this, we’re going to use a tool called Postman with a pre-made collection of endpoints.

If you’ve never used Postman before, check out our [Testing Conduit Apps Using Postman](#) guide.

Open up Postman and use the “Register” request inside the “Auth” folder to create a new user.

{x: make your first postman request}
Make your first Postman request.

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

{x: create authentication renderers file} 
Create `conduit/apps/authentication/renderers.py`.

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

{x: update userretrieveupdateapiview with userjsonrenderer} 
Update `UserRetrieveUpdateAPIView` to use the `UserJSONRenderer` renderer class.

With `UserJSONRenderer` in place, go ahead and use the “Register” Postman request to create a new user. Notice how, this time, the response is inside the “user” namespace.

{x: register a new user with postman 001}
Register a new user with Postman.

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

{x: add loginserializer to conduit/apps/authentication/serializers.py}
Add the new `LoginSerializer` class to `conduit/apps/authentication/serializers.py`.

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

{x: add loginapiview to conduit/apps/authentication/views.py}
Add the new `LoginAPIView` class to `conduit/apps/authentication/views.py`.

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

{x: add a url pattern for loginapiview}
Add a URL pattern for `LoginAPIView` to `conduit/apps/authentication/urls.py`.

### Logging a user in with Postman

At this point, a user should be able to log in by hitting the new login endpoint. Let’s test this out. Open up Postman and use the “Login” request to log in with one of the users you created previously. If the login attempt was successful, the response will include a token that can be used in the future when making requests that require the user be authenticated.

{x: log a user in with postman 001}
Log a user in using Postman.

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

{x: create conduit/apps/core/exceptions.py}
Create `conduit/apps/core/exceptions.py` with the above code.

With that taken care of, open up `conduit/settings.py` and add a new setting to the bottom of the file called `REST_FRAMEWORK`, like so:

```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'conduit.apps.core.exceptions.core_exception_handler',
    'NON_FIELD_ERRORS_KEY': 'error',
}
```

{x: update settings with exception_handler and non_field_errors_key}
Add the `REST_FRAMEWORK` dict to `conduit/settings.py` with the `EXCEPTION_HANDLER` and `NON_FIELD_ERRORS_KEY` keys as above.

This is how we override settings in DRF. We will add one more setting in a bit when we start writing views that require the user to be authenticated.

Let’s try sending another login request using Postman. Be sure to use an email/password combination that is invalid.

{x: use postman to log in with an invalid username and password}
Use Postman to log in with an invalid username and password. Make sure the response is formatted as we discussed earlier.

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

{x: update userjsonrenderer to handle errors key}
Update `UserJSONRenderer` in `conduit/apps/authentication/renderers.py` to handle the case where `data` contains an `errors` key.

Now send the login request with Postman one more time and all should be well.

{x: use postman to log in with an invalid username and password 002}
Use Postman to log in with an invalid username and password. Make sure the response has a `users` key instead of an `errors` key.

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

{x: add the userserializer class to conduit/apps/authentication/serializers.py}
Add the new `UserSerializer` class to `conduit/apps/authentication/serializers.py`.

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

{x: add the new userretrieveupdateapiview class to conduit/apps/authentication/views.py}
Add the new `UserRetrieveUpdateAPIView` class to `conduit/apps/authentication/views.py`.

Now go over to `conduit/apps/authentication/urls.py` and update the imports to include `UserRetrieveUpdateAPIView`:

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

{x: add a url pattern for userretrieveupdateapiview in conduit/apps/authentication/urls.py}
Add a URL pattern for `UserRetrieveUpdateAPIView` to `conduit/apps/authentication/urls.py`.

Open up Postman again and send the “Current User” request. You should get an error with a response that looks like this:

```json
{
  "user": {
    "detail": "Authentication credentials were not provided."
  }
}
```

{x: use postman to request the current user and get an error}
Use Postman to request data on the current user. You should get an error saying `Authentication credenetials were not provided`.

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

{x: check out the pyjwt docs}
There isn’t any extra reading to do here, but feel free to check out the docs for the [PyJWT](https://pyjwt.readthedocs.io/en/latest/) library, if you’re interested.

### Telling DRF about our authentication backend

We must explicitly tell Django REST Framework which authentication backend we want to use, similar to how we told Django to use our custom User model.

Open up `conduit/settings.py` and update the `REST_FRAMEWORK` dict with a new key:

```diff
REST_FRAMEWORK = {
    # ...
+
+    'DEFAULT_AUTHENTICATION_CLASSES': (
+        'authentication.backends.JWTAuthentication',
+    ),
}
```

{x: register our jwt authentication backend} 
Register our JWT authentication backend with Django REST Framework by adding `DEFAULT_AUTHENTICATION_CLASSES` to the `REST_FRAMEWORK` setting in `conduit/settings.py`.

## Retrieving and updating users with Postman

With our new authentication backend in place, the authentication error we saw a while ago should be gone. Test this by opening Postman and sending another “Current User” request. The request should be successful and you should see the information about your user in the response.

{x: request the current user with postman}
Use Postman to request the current user. This time the request should be successful.

Remember that we created an update endpoint at the same time we created the retrieve endpoint. Let’s test this one too. Send the request labeled “Update User” in the “Auth” folder in Postman. If you used the default body, the email of your user should have changed. Feel free to play around with these requests to make changes to your user.

{x: use postman to update the current user 001}
Use Postman to update the current user. Play around with this request to make changes to your user.

## On to better things

That’s all there is for this chapter. We’ve created a user model and serialized users in three different ways. There are four shiny new endpoints that let users register, login, and retrieve and update their information. We’re off to a really good start here!

Next up, we’re going to create profiles for our users. You may have noticed that the `User` model is pretty bare-bones. We only included things essential for authentication. Other information such as a biography and avatar URL will go in the `Profile` model that we’ll work with in the next chapter.