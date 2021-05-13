# Bootstrapping Django app with Cognito

#### Introduction

![](https://raw.githubusercontent.com/glebpushkov/articles/master/bootstrapping_django_app_with_cognito/images/1.jpeg)

Nowadays, more and more developers integrate their app with Single sign-on (SSO) services. It greatly increases the speed of development, because the basic routine is already implemented, tested, and hosted: sign-in, registration, reset password, 2FA, email and phone verification, welcome-email, and other. It's especially useful if you have a microservice architecture - once user authorized - he can send requests to any of them. 

There are several solutions that offer such services, like Azure Active Directory B2C, AWS Cognito, Okta, Auth0, and others. They differ by functionality (ensure it meets your app needs, like multilanguage emails), and especially, by price.

AWS Cognito is the cheapest one (but be aware that using lambdas, 2FA, SNS could additionally generate associated costs which might not be originally mentioned). Also, it's very flexible - you can [customize workflow with Lambda Triggers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html) (during authorization events your custom AWS Lambda functions could be called where you free to do whatever you want), or even create your own [Authentication Challenges](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-challenge.html). If your application stack is hosted on AWS and managed via CloudFormation (or Terraform) - it's also handy to set up and configure Cognito as an additional resource of your IaC (but worth to be aware of some cautions mentioned after Django integration part).

According to [documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-with-identity-providers.html) after successful authentication Amazon Cognito API returns `id_token`, `access_token` and `refresh_token`. Both `id_token` and `access_token` are JSON Web Tokens and could be used to identify a user during API requests to the Django application. The `id_token` contains personal identity information such as `name`, `email`, and `phone_number`. The `access_token` doesn't carry such information, so it's more secure to use it as JWT encoded via base64, and everybody who gets this token can easily see personal data. Even if you have encrypted https connection, there are ways to steal the data, like `SSL Stip` (hint - configure HSTS headers!).

On high-view level the authentication process will look like that:

![](https://raw.githubusercontent.com/glebpushkov/articles/master/bootstrapping_django_app_with_cognito/images/2.png)

## Cognito and Django

(Before going to deep dive into the guide, please check currently available libraries for that, and how flexible they are. Probably they may fit your requirements. Still, you may find useful details which highlight how it works, and what to expect if you use Cognito)

In the current guide we will cover a case when we want to have two types of users - backoffice users (to login and work with django-admin, session authorization) and application users (to interact with api endpoints, such users registered in Cognito, jwt-authorization). 

#### 1. Install packages

I found several libraries that help with integration, but it's not hard to build flexible and easy-to-extend configuration on our own using DRF djangorestframework-jwt. 

***NOTE:** The original library is no longer [maintained](https://github.com/jpadilla/django-rest-framework-jwt/issues/484), so we will use a [fork](https://github.com/Styria-Digital/django-rest-framework-jwt) of it (drf-jwt). Another [alternative library for jwt](https://github.com/SimpleJWT/django-rest-framework-simplejwt) is currently also looking for a maintainer.*

`pip install djangorestframework cryptography drf-jwt`

#### 2. Create a User Pool in AWS Cognito

Sign-in into your AWS console and proceed to Cognito. Press `Manage User Pools` (the `Identity pool` is something [different](https://aws.amazon.com/premiumsupport/knowledge-center/cognito-user-pools-identity-pools/)). Create new user pool and configure attributes.

NOTE: once you set up required attrs - you wouldn't be able to change them without re-creating a pool and losing all users' passwords. Btw, you have to migrate your users on your own. More thoughts about that at the end of the article. I recommend having as few required attributes as possible. For example - only email.

Create two app clients. One for a frontend app - `Enable username-password (non-SRP) flow for app-based authentication (USER_PASSWORD_AUTH)` . If you will need to access user pool data from the backend app - add one more client `Enable sign-in API for server-based authentication (ADMIN_NO_SRP_AUTH)`.

For easy testing of integration, you can enable "Hosted UI" - Sign-Up/Sign-In pages are provided by Cognito, so you can easily obtain JWT tokens and use them in Postman to ensure the configuration of Django side was done in a proper way.

![](https://raw.githubusercontent.com/glebpushkov/articles/master/bootstrapping_django_app_with_cognito/images/3.png)

To do this you need to specify **Amazon Cognito domain** in "Domain name" section, e.g.:

https://any-fancy-name-you-like.auth.eu-central-1.amazoncognito.com

in "App client settings" you need to enable any of "OAuth Flows" (let's say `Implicit grant`) and at least "OAuth Scope" ( `openid` ). Also provide a callback URL - http://localhost:8000/admin.

After saving your changes, and the bottom of the same page you will see "Launch Hosted UI" link - it will lead you to login form. Create a user in the "Users and groups" tab and use its credentials to log in via Hosted UI. 

<img hosted-ui.png>

As a result, a browser will be redirected to callback URL which will have desired tokens:

http://localhost:8000/admin#id_token=eyJraWQiOiJNdm...&access_token=eyJraWQiOiJqenIwdnRVK....&expires_in=3600&token_type=Bearer

congrats, we obtained `id_token` and `access_token`!

to see what's inside - go to https://jwt.io/ and put the token into debugger - for `access_token` payload would be 

```json
{
  "sub": "9387dbed-ce4a-44fa-b6ab-6b26327e9305",
  "event_id": "01c908b7-c15c-42b4-9849-855af5528051",
  "token_use": "access",
  "scope": "openid email",
  "auth_time": 1572171426,
  "iss": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_bUFIXsrqe",
  "exp": 1572175026,
  "iat": 1572171426,
  "version": 2,
  "jti": "cc5eeb1d-686e-42ad-b65d-5c5583a80140",
  "client_id": "2bdgd681nmmnickj0coq0j1oq1",
  "username": "9387dbed-ce4a-44fa-b6ab-6b26327e9305"
}
```

#### 3. Create custom User model

It's a good practice to override the default user model once you start building your Django app, otherwise, it's painful to migrate on a [mid-project phase](https://docs.djangoproject.com/en/2.2/topics/auth/customizing/#changing-to-a-custom-user-model-mid-project).

Side-note: I would recommend to setup an abstract base model which would be used everywhere. It can be placed in the special app where general and non-related to business logic application code lives, for example - `core`

```python
import uuid

from django.db import models


class AbstractBaseModel(models.Model):
    """
    Base abstract model, that has `uuid` instead of `id` and includes `created_at`, `updated_at` fields.
    """
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False, unique=True)
    created_at = models.DateTimeField('Created at', auto_now_add=True)
    updated_at = models.DateTimeField('Updated at', auto_now=True)

	class Meta:
        abstract = True
    
    def __repr__(self):
        return f'<{self.__class__.__name__} {self.id}>'

```

Why it's helpful:

1. `uuid4` as a primary key - 

   identifiers like 91eea07d-0742-4925-9c39-fb6d6e352f2a would be generated instead of regular incremental 12, 13, 14. Nobody will know how many Orders or Users do you have. Also, `uuid4` is generated on Python-side, so you can know a PK before insertion. There are many discussions about the size, performance and other disadvantages, one of the good articles I found is https://tomharrisonjr.com/uuid-or-guid-as-primary-keys-be-careful-7b2aa3dcb439. From my experience, `uuid4`is really neat and it gives additional flexibility - for example if you don't know which table to lookup - you can query multiple of them until you get the desired row since UUID would be unique across the whole database;

2. `created_at` and `updated_at` - useful fields which will be added to all models;

3. Informative `__repr__` - extreemly helpful if you're using [Sentry](https://sentry.io/) to collect errors and exceptions - in the stack trace you will see `<User 75145554-4142-480b-bcfa-36840b315ba1>` instead of `<object at 0x1028ed080>`

4. Such a layer gives you ability in a transparent way to override the behavior of all your models, in the way like we just did with `__repr__` or with base fields.

Now let's create a custom user at `account/models.py`:

```python
from django.db import models

from django.contrib.auth.base_user import AbstractBaseUser
from django.contrib.auth.models import PermissionsMixin
from django.contrib.auth.validators import UnicodeUsernameValidator

from core.models import AbstractBaseModel


class User(PermissionsMixin, AbstractBaseUser, AbstractBaseModel):
    """
    Table contains cognito-users & django-users.
    PermissionsMixin leverage built-in django model permissions system
    (which allows to limit information for staff users via Groups).
    Note: Django-admin user and app user not split in different tables because of simplicity of development.
    Some libraries assume there is only one user model, and they can't work with both.
    For example to have a history log of changes for entities - to save which user made a change of object attribute,
    perhaps, auth-related libs, and some other.
    With current implementation we don't need to fork, adapt and maintain third party packages.
    They should work out of the box.
    The disadvantage is - cognito-users will have unused fields which always empty. Not critical.
    """
    username_validator = UnicodeUsernameValidator()

    ### Common fields ###
    # For cognito-users username will contain `sub` claim from jwt token
    # (unique identifier (UUID) for the authenticated user).
    # For django-users it will contain username which will be used to login into django-admin site
    username = models.CharField('Username', max_length=255, unique=True, validators=[username_validator])
    is_active = models.BooleanField('Active', default=True)

    ### Cognito-user related fields ###
    # some additional fields which will be filled-out only for users registered via Cognito
    pass

    ### Django-user related fields ###
    # password is inherited from AbstractBaseUser
    email = models.EmailField('Email address', blank=True)  # allow non-unique emails
    is_staff = models.BooleanField(
        'staff status',
        default=False,
        help_text='Designates whether the user can log into this admin site.'
    )

    USERNAME_FIELD = 'username'
    EMAIL_FIELD = 'email'
    REQUIRED_FIELDS = ['email']  # used only on createsuperuser

    @property
    def is_django_user(self):
        return self.has_usable_password()
```

And in `settings.py` change a default user model

```python
AUTH_USER_MODEL = 'account.User'
```

Now we can run migrations.

And the final step - let's register custom model in `account/admin.py`:

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from django.contrib.auth.forms import UserCreationForm, UserChangeForm, UsernameField
from django.utils.translation import ugettext_lazy as _

from account.models import User


class CustomUserCreationForm(UserCreationForm):

    class Meta(UserCreationForm.Meta):
        model = User


class CustomUserChangeForm(UserChangeForm):

    class Meta(UserCreationForm.Meta):
        model = User
        fields = '__all__'
        field_classes = {'username': UsernameField}


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    fieldsets = (
        (None, {'fields': ('username', 'email', 'password', )}),
        (
            _('Permissions'),
            {'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions', )}
        ),
        (_('Important dates'), {'fields': ('created_at', 'updated_at', )}),
    )
    readonly_fields = ('created_at', 'updated_at', )
    add_fieldsets = (
        (None, {
            'classes': ('wide', ),
            'fields': ('username', 'email', 'password1', 'password2', ),
        }),
    )
    form = CustomUserChangeForm
    add_form = CustomUserCreationForm
    list_display = ('username', 'is_staff', 'is_active', )
```

#### 4. Configure REMOTE_USER

In case when external authentication sources are used - additional configuration has to be done https://docs.djangoproject.com/en/3.1/howto/auth-remote-user/. The `RemoteUserBackend` creates a new User record in the database if it can't find existing. This behavior can be changed by `create_unknown_user`, find more info in the docs https://docs.djangoproject.com/en/3.1/ref/contrib/auth/#django.contrib.auth.backends.RemoteUserBackend. 

```python
MIDDLEWARE = [
    ...
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.RemoteUserMiddleware',
	...
]

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.RemoteUserBackend',
    'django.contrib.auth.backends.ModelBackend',
]
```

#### 5. Configure DRF

`settings.py` 

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'core.api.permissions.DenyAny',
    ),
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
    ),
}
```

Bonus advice:

To reduce the number of security issues that could happen due to inattentiveness I would recommend to override the default permission class. In `core/api/permissions`you can put the following class:

```python
class DenyAny(BasePermission):
    def has_permission(self, request, view):
        return False

    def has_object_permission(self, request, view, obj):
        return False
```

`settings.py`

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'core.api.permissions.DenyAny',
    ),
    ...
}
```

#### 6. Configure djangorestframework-jwt

On application start we need to download public JWKS which will be used to verify JWT. 

`settings.py`

```python
import json
from urllib import request
...

COGNITO_AWS_REGION = 'eu-central-1'
COGNITO_USER_POOL = 'eu-central-1_xxxxxx'
# Provide this value if `id_token` is used for authentication (it contains 'aud' claim).
# `access_token` doesn't have it, in this case keep the COGNITO_AUDIENCE empty
COGNITO_AUDIENCE = None
COGNITO_POOL_URL = None  # will be set few lines of code later, if configuration provided

rsa_keys = {}
# To avoid circular imports, we keep this logic here.
# On django init we download jwks public keys which are used to validate jwt tokens.
# For now there is no rotation of keys (seems like in Cognito decided not to implement it)
if COGNITO_AWS_REGION and COGNITO_USER_POOL:
    COGNITO_POOL_URL = 'https://cognito-idp.{}.amazonaws.com/{}'.format(COGNITO_AWS_REGION, COGNITO_USER_POOL)
    pool_jwks_url = COGNITO_POOL_URL + '/.well-known/jwks.json'
    jwks = json.loads(request.urlopen(pool_jwks_url).read())
    rsa_keys = {key['kid']: json.dumps(key) for key in jwks['keys']}


JWT_AUTH = {
    'JWT_PAYLOAD_GET_USERNAME_HANDLER': 'core.api.jwt.get_username_from_payload_handler',
    'JWT_DECODE_HANDLER': 'core.api.jwt.cognito_jwt_decode_handler',
    'JWT_PUBLIC_KEY': rsa_keys,
    'JWT_ALGORITHM': 'RS256',
    'JWT_AUDIENCE': COGNITO_AUDIENCE,
    'JWT_ISSUER': COGNITO_POOL_URL,
    'JWT_AUTH_HEADER_PREFIX': 'Bearer',
}
```

(In a real project settings have to be less hardcoded, something like that

```python
COGNITO_AWS_REGION = env('COGNITO_AWS_REGION', default=None)
```

There is already a nice article about how to [manage django settings as a **pro** by Alexander Ryabtsev](https://djangostars.com/blog/configuring-django-settings-best-practices/) which worth to checkout).

Downloaded rsa keys look like that - 

```json
{
  "keys": [
    {
      "alg": "RS256",
      "e": "AQAB",
      "kid": "Mvd6BSFCvQ+PbEOQCqOZd3CCSdd/d/mw+65R5uN1+r0=",
      "kty": "RSA",
      "n": "kQgIEUZBMkoN7jU_rRxjH...B1tcoSa4EkYUZtDsQ",
      "use": "sig"
    },
    {
      "alg": "RS256",
      "e": "AQAB",
      "kid": "jzr0vtU+c+hY2apVvODttwoYVSpdS/Bhn8D7YLAXe7o=",
      "kty": "RSA",
      "n": "l6m0rB8RSQWmp8gijxjYK...Na77QY8cRfzNLuLmzw",
      "use": "sig"
    }
  ]
}
```

If you check a decoded id_token` and `access_token in their headers there are

```json
{
  "kid": "Mvd6BSFCvQ+PbEOQCqOZd3CCSdd/d/mw+65R5uN1+r0=",
  "alg": "RS256"
}
```

that specifies which JWKS should be used to validate a token. The "mapping" logic has to be implemented by our own in cognito_jwt_decode_handler:

`core/utils/jwt`

```python
import jwt
from jwt import DecodeError
from jwt.algorithms import RSAAlgorithm

from rest_framework_jwt.settings import api_settings

from django.contrib.auth import authenticate


def get_username_from_payload_handler(payload):
    username = payload.get('sub')
    authenticate(remote_user=username)
    return username


def cognito_jwt_decode_handler(token):
    """
    To verify the signature of an Amazon Cognito JWT, first search for the public key with a key ID that
    matches the key ID in the header of the token. (c)
    https://aws.amazon.com/premiumsupport/knowledge-center/decode-verify-cognito-json-token/
    Almost the same as default 'rest_framework_jwt.utils.jwt_decode_handler', but 'secret_key' feature is skipped
    """
    options = {'verify_exp': api_settings.JWT_VERIFY_EXPIRATION}
    unverified_header = jwt.get_unverified_header(token)
    if 'kid' not in unverified_header:
        raise DecodeError('Incorrect authentication credentials.')

    kid = unverified_header['kid']
    try:
        # pick a proper public key according to `kid` from token header
        public_key = RSAAlgorithm.from_jwk(api_settings.JWT_PUBLIC_KEY[kid])
    except KeyError:
        # in this place we could refresh cached jwks and try again
        raise DecodeError('Can\'t find proper public key in jwks')
    else:
        return jwt.decode(
            token,
            public_key,
            api_settings.JWT_VERIFY,
            options=options,
            leeway=api_settings.JWT_LEEWAY,
            audience=api_settings.JWT_AUDIENCE,
            issuer=api_settings.JWT_ISSUER,
            algorithms=[api_settings.JWT_ALGORITHM]
        )
```

Usually, JWKS should be periodically rotated by auth service and you could have a `KeyError` which indicates that you need to download new JWKS keys, but Cognito doesn't have this functionality.

A few notes about  `get_username_from_payload_handler` - `sub` in jwt payload carries unique UUID of authenticated user in Cognito User Pool. You can use it as pk for your users in a database, or, to be more independent - keep your own-generated uuid4 as pk for users, and store Cognito`sub` as `username` (it's what we're doing now). But be aware that this way could add complexity to your implementation if a client/frontend app works directly with Cognito and 'knows' only `sub`, but not internal uuid4 of a user, which would be used in DB to build relations between models.

### 7. Create test view

`account/api/serializers.py`

```python
from rest_framework import serializers

from account.models import User


class UserSerializer(serializers.ModelSerializer):
    """ Used to retrieve user info """

    class Meta:
        model = User
        fields = '__all__'
```

`account/api/views.py`

```python
from rest_framework.generics import GenericAPIView
from rest_framework.mixins import RetrieveModelMixin
from rest_framework.permissions import IsAuthenticated

from account.api.serializers import UserSerializer


class UserProfileAPIView(RetrieveModelMixin, GenericAPIView):
    serializer_class = UserSerializer
    permission_classes = (IsAuthenticated, )

    def get_object(self):
        return self.request.user

    def get(self, request, *args, **kwargs):
        """
        User profile
        Get profile of current logged in user.
        """
        return self.retrieve(request, *args, **kwargs)
```

`urls.py`

```python
from account.api.views import UserProfileAPIView


urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/me', UserProfileAPIView.as_view(), name='my_profile'),
]
```

#### 8. Run server and make a request!

We will use a [Postman](https://www.getpostman.com/) to ensure integration code works as it should.

Create a new GET request to http://localhost:8000/api/v1/me and in Headers provide `Authorization` with value `Bearer <access_token>`. The token is valid for 1 hour, so you may need to obtain a new one via Hosted UI (step 3).

Congratulations, everything is done!

(Full project sample may be found at my [GitHub](https://github.com/glebpushkov/django-cognito-example)).

#### Debugging

Sometimes it's hard, and it's better to dig into the source code and check when and why any of the messages appear. For example `{"detail": "Invalid signature."}`actually means that there is no such user in the database - which indicates that `REMOTE_USER` was not properly configured as it has to create a new user if it fails to find existing one.

#### What’s Worth to Note when Working with AWS Cognito:

Below, I’ve listed a number of facts that I’ve faced when working with Cognito. They aren’t either pros or cons. These are just peculiarities you should know when starting to develop a Django app authentication feature with Cognito and tips how to solve them.

- Update: there is a new client configuration 'Prevent User Existence Errors' which solves the issue. 
  If this configuration is not set (or you have old cognito pool) Sing-in and reset password API explicitly indicate that the user with provided email hasn't registered: `{__type: "UserNotFoundException", message: "User does not exist."}` - by default anyone can test a list of emails and identify who's already registered in your application. 
  Worth to mention that Cognito has built-in protection for that, but there are not much details how it works - e.g is it only for sign-in api, or reset password is protected too... To protect registration form - you may think about adding a captcha - seems like it's possible, but there is no solution out of the box - it requires configuring custom auth lambda functions flow.
- The things have changed while this article was draft - finally ["Amazon Cognito User Pools service now supports case insensitivity for user aliases"](https://aws.amazon.com/about-aws/whats-new/2020/02/amazon-cognito-user-pools-service-now-supports-case-insensitivity-for-user-aliases/). It's a new feature for **new** User Pools, but existing still have such problem. I was very surprised when we find out that both usernames and emails were treated as case-sensitive. So “JohnSmith@exmaple.com” is different than “johnsmith@exmaple.com,” who is different than “jOhNSmiTh@exmaple.com”. One option to fix that was: in frontend app makes email lowercase. Another intention was to fix that on Cognito Pool level with lambda function on "PreSignUp" trigger, but it doesn't allow to change the input values - so the only way is to override email/username after a user was created in "PostConfirmation" lambda, which is not so elegant;
- Depending on the type of application you can face a problem when you need to display a list of users with name/picture/phone/other-field - in such case, you need to retrieve users from Cognito each time when data is requested, or, otherwise, implement profiles sync logic and store copy of all Cognito users in your database. It's common to any SSO service, but it's connected to the next point you may face; 
- No lambda trigger on attribute update. If a user modify his profile data via Cognito API there is no callback which indicates that data have been changed. If you store a copy of Cognito data in you db (for convenience) you have to use some workarounds like: fronted code have to notify your services explicitly when user data in Cognito was successfully updated  - and then you can pull the changes from User Pool;
- Cognito allows the creation of multiple users with the same email until one of them becomes verified (might happen when the user double-click the submit button of the registration form). That's ok, but you have to cover this case when working with User Pool via code/scripts;
- No export users functionality. There is an npm package which can do that, or you need to implement this on your own;
- You can't export users' password hashes (in the case if you want to change SSO provider or use your own auth services, or even if you want to move users from one User Pool to another). You will have to send an email asking them to set a new password and ensure you show explanation and support a flow when users will start trying to login with their passwords;
- During signup flow users have always to enter login & password right after clicking on the confirmation link in the email. There is no way to auto-login the user. A lot of complaints, but still this hasn't been improved by Cognito;
- Default fields phone_verified and email_verified may store 'True' or 'true' values. Strange? - yes.
- No easy way to mark token as invalid when user changes password or signs out https://github.com/aws-amplify/amplify-js/issues/3435
- And a few more things have been already covered [here](https://securityboulevard.com/2019/02/cave-of-broken-mirrors-3-issues-with-aws-cognito/).

#### Cognito & CloudFormation

CloudFormation is a great tool that allows you to store & maintain your infrastructure as a code. You can define multiple stacks, for each of stack you need to create a special JSON template that defines your resources and configuration. Such files could be managed manually, or they could be generated by [Troposphere](https://github.com/cloudtools/troposphere) via Python code (still keep in mind that for growing infrastructure it's easy to overengineer and have a hard maintainable codebase, which will hurt a lot in a long-term). CloudFormation automatically rollbacks all changes if something went wrong during a stack update, which is really great feature. But the way how it manages Cognito is a bit strange, thus I would like to highlight the most important notes:

1) If you created User Pool manually (as any other type of resources) - it's impossible to add it to CloudFormation stack without re-creation. CloudFormation can only manage resources that were created by CloudFormation. For "stateless" resources, like EC2, it's not a problem, for RDS it's trickier as will require to restore db from a snapshot to not lose a data, but with Cognito, it's not possible to easily migrate as it doesn't allow to export passwords - this information will be lost and users will need to set up passwords one more time. Still, in such situation User Pool can be kept unmanaged, but you can use a reference (`Ref`) in the template to connect it with rest of your stack;

2) **CloudFormation can drop User Pool and all data you have inside**. That's my own experience which happened in summer 2019 on our dev environment (also mentioned in a referenced [article](https://securityboulevard.com/2019/02/cave-of-broken-mirrors-3-issues-with-aws-cognito/) above). Still, when I started work on the article half a year later and tried to reproduce the scenario (change the ordering of attributes in a template, or add new) - it didn't happen this time. Likely this was fixed but I can't guarantee. Actually, this was the main driver for me to share experience working with this stack. Some advice to protect yourself: 

- ensure you provided [DeletionPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) and even if something bad will happen - the resource will be gone from a stack (would be still accessible from a template using `Ref`), but it wouldn't be physically removed;
- on each stack update before applying a new template carefully review change set. Search there for a User Pool resource name and for words like "changeSource": "DirectModification" on this resource. That might be a flag that something might go wrong, depending on the changes. If AWS really fixed the issue, this point is not valid anymore, but personally I have a Vietnam Syndrome once I see that.

Not sure how does Terraform or other tools deal with Cognito, but you have to be **extreeeeeeemly** careful here, otherwise you can accidentally kill the product.

#### Summary

![](https://raw.githubusercontent.com/glebpushkov/articles/master/bootstrapping_django_app_with_cognito/images/4.png)

Picture yourself as a barber and SSO service as a straight razor. It’s a great tool if you know how to deal with it, but otherwise the risk of making something wrong (with really bad consequences) is very high. It’s a tool that you have to pick wisely and configure carefully, put a lot of effort into investigating how it works, and even its worth to spend time building a proof of concept to get a better feeling of its abilities and restrictions. Once you release an application and get first users, there is no way back: migration to another service will be very painful with import/export of user passwords being a slippy thing. If you have anything to add, highlight some pitfalls, or recommend something around SSO integration, please leave a comment, and one day it will help someone to build a successful story.

