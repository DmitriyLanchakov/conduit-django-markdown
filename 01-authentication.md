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

{x: read up on manager classes} Read […](#) to learn more about what a `Manager` class is.

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
Read about [`ModelSerializer`](#).

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

That’s the only view we’re going to write for now. Let’s expose `RegistrationAPIView` to our clients by adding it to the list of URLs.

Make a file called `conduit/apps/authentication/urls.py` with the following content:

```python
from django.conf.urls import url

from .views import RegistrationAPIView

urlpatterns = [
    url(r'^users/?$', RegistrationAPIView.as_view()),
]
```

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

{x: read up on manager classes} Read […](#) to learn more about what a `Manager` class is.

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
Read about [`ModelSerializer`](#).

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

That’s the only view we’re going to write for now. Let’s expose `RegistrationAPIView` to our clients by adding it to the list of URLs.

Make a file called `conduit/apps/authentication/urls.py` with the following content:

```python
from django.conf.urls import url

from .views import RegistrationAPIView

urlpatterns = [
    url(r'^users/?$', RegistrationAPIView.as_view()),
]
```

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
