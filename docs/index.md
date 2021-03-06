### DRF-extensions

DRF-extensions is a collection of custom extensions for [Django REST Framework](https://github.com/tomchristie/django-rest-framework).
Source repository is available at [https://github.com/chibisov/drf-extensions](https://github.com/chibisov/drf-extensions).


### Viewsets

Extensions for [viewsets](http://django-rest-framework.org/api-guide/viewsets.html).

#### DetailSerializerMixin

This mixin lets add custom serializer for detail view. Just add mixin and specify `serializer_detail_class` attribute:

    from django.contrib.auth.models import User
    from myapps.serializers import UserSerializer, UserDetailSerializer
    from rest_framework_extensions.mixins import DetailSerializerMixin

    class UserViewSet(DetailSerializerMixin, viewsets.ReadOnlyModelViewSet):
        serializer_class = UserSerializer
        serializer_detail_class = UserDetailSerializer
        queryset = User.objects.all()

Sometimes you need to set custom QuerySet for detail view. For example, in detail view you want to show user groups and permissions for these groups. You can make it by specifying `queryset_detail` attribute:

    from django.contrib.auth.models import User
    from myapps.serializers import UserSerializer, UserDetailSerializer
    from rest_framework_extensions.mixins import DetailSerializerMixin

    class UserViewSet(DetailSerializerMixin, viewsets.ReadOnlyModelViewSet):
        serializer_class = UserSerializer
        serializer_detail_class = UserDetailSerializer
        queryset = User.objects.all()
        queryset_detail = queryset.prefetch_related('groups__permissions')

If you use `DetailSerializerMixin` and don't specify `serializer_detail_class` attribute, then `serializer_class` will be used.

If you use `DetailSerializerMixin` and don't specify `queryset_detail` attribute, then `queryset` will be used.


#### PaginateByMaxMixin

*New in DRF-extensions Development version*

This mixin allows to paginate results by [max\_paginate\_by](http://www.django-rest-framework.org/api-guide/pagination#pagination-in-the-generic-views)
value. This approach is useful when clients want to take as much paginated data as possible,
but don't want to bother about backend limitations.

    from myapps.serializers import UserSerializer
    from rest_framework_extensions.mixins import PaginateByMaxMixin

    class UserViewSet(PaginateByMaxMixin,
                      viewsets.ReadOnlyModelViewSet):
        max_paginate_by = 100
        serializer_class = UserSerializer

And now you can send requests with `?page_size=max` argument:

    # Request
    GET /users/?page_size=max HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        count: 1000,
        next: "https://localhost:8000/v1/users/?page=2&page_size=max",
        previous: null,
        results: [
            ...100 items...
        ]
    }

This mixin could be used only with Django Rest Framework >= 2.3.8, because
[max\_paginate\_by](http://www.django-rest-framework.org/topics/release-notes#238)
was introduced in 2.3.8 version.


#### Cache/ETAG mixins

**ReadOnlyCacheResponseAndETAGMixin**

This mixin combines `ReadOnlyETAGMixin` and `CacheResponseMixin`. It could be used with
[ReadOnlyModelViewSet](http://www.django-rest-framework.org/api-guide/viewsets.html#readonlymodelviewset) and helps
to process caching + etag calculation for `retrieve` and `list` methods:

    from myapps.serializers import UserSerializer
    from rest_framework_extensions.mixins import (
        ReadOnlyCacheResponseAndETAGMixin
    )

    class UserViewSet(ReadOnlyCacheResponseAndETAGMixin,
                      viewsets.ReadOnlyModelViewSet):
        serializer_class = UserSerializer

**CacheResponseAndETAGMixin**

This mixin combines `ETAGMixin` and `CacheResponseMixin`. It could be used with
[ModelViewSet](http://www.django-rest-framework.org/api-guide/viewsets.html#modelviewset) and helps
to process:

* Caching for `retrieve` and `list` methods
* Etag for `retrieve`, `list`, `update` and `destroy` methods

Usage:

    from myapps.serializers import UserSerializer
    from rest_framework_extensions.mixins import CacheResponseAndETAGMixin

    class UserViewSet(CacheResponseAndETAGMixin,
                      viewsets.ModelViewSet):
        serializer_class = UserSerializer

Please, read more about [caching](#caching), [key construction](#key-constructor) and [conditional requests](#conditional-requests).


### Routers

Extensions for [routers](http://django-rest-framework.org/api-guide/routers.html).

You will need to use custom `ExtendedDefaultRouter` or `ExtendedSimpleRouter` for routing if you want to take advantages of described extensions. For example you have standard implementation:

    from rest_framework.routers import DefaultRouter
    router = DefaultRouter()

You should replace `DefaultRouter` with `ExtendedDefaultRouter`:

    from rest_framework_extensions.routers import (
        ExtendedDefaultRouter as DefaultRouter
    )
    router = DefaultRouter()

Or `SimpleRouter` with `ExtendedSimpleRouter`:

    from rest_framework_extensions.routers import (
        ExtendedSimpleRouter as SimpleRouter
    )
    router = SimpleRouter()

#### Collection level controllers

Out of the box Django Rest Framework has controller functionality for detail views. For example:

    from django.contrib.auth.models import User
    from rest_framework import viewsets
    from rest_framework.decorators import action, link
    from rest_framework.response import Response
    from myapp.serializers import UserSerializer

    class UserViewSet(viewsets.ModelViewSet):
        """
        A viewset that provides the standard actions
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

        @action()
        def set_password(self, request, pk=None):
            return Response(['password changed'])

        @link()
        def groups(self, request, pk=None):
            return Response(['user groups'])

Change password request:

    # Request
    POST /users/1/set_password/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['password changed']

User groups request:

    # Request
    GET /users/1/groups/ HTTP/1.1
    Accept: application/json
    
    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['user groups']

But what if you want to add custom controller to collection level? 

DRF-extensions `action` and `link` decorators will help you with it. These decorators behaves exactly as default, but can receive additional parameter `is_for_list`:

    ...
    from rest_framework_extensions.decorators import action, link
    ...

    class UserViewSet(viewsets.ModelViewSet):
        """
        A viewset that provides the standard actions
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

        @action(is_for_list=True)
        def confirm_email(self, request, pk=None):
            return Response(['email confirmed'])

        @link(is_for_list=True)
        def surname_first_letters(self, request, pk=None):
            return Response(['a', 'b', 'c'])

        @action()
        def set_password(self, request, pk=None):
            return Response(['password changed'])

        @link()
        def groups(self, request, pk=None):
            return Response(['user groups'])

Now you can post to collection level controller:

    # Request
    POST /users/confirm_email/ HTTP/1.1
    Accept: application/json
    Content-Type: application/json
    
    {"code": 123456}
    
    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['email confirmed']

Or retrieve data from link collection level controller:

    # Request
    GET /users/surname_first_letters/ HTTP/1.1
    Accept: application/json
    
    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['a', 'b', 'c']

#### Controller endpoint name

By default function name will be used as name for url routing:

    # Reqeuest
    POST /users/1/set_password/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['password changed']

But what if you want to specify custom name in url. For example `set-password`.

DRF-extensions `action` and `link` decorators will help you to specify custom endpoint name.
These decorators behaves exactly as default, but can receive additional parameter `endpoint`:

    ...
    from rest_framework_extensions.decorators import action, link
    ...

    class UserViewSet(viewsets.ModelViewSet):
        """
        A viewset that provides the standard actions
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

        @action(endpoint='set-password')
        def set_password(self, request, pk=None):
            return Response(['password changed'])

Change password request:

    # Request
    POST /users/1/set-password/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['password changed']


### Fields

Set of serializer fields that extends [default fields](http://www.django-rest-framework.org/api-guide/fields) functionality.

#### ResourceUriField

Represents a hyperlinking uri that points to the detail view for that object.

    from rest_framework_extensions.fields import ResourceUriField

    class CitySerializer(serializers.ModelSerializer):
        resource_uri = ResourceUriField(view_name='city-detail')

        class Meta:
            model = City

Request example:

    # Request
    GET /cities/268/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
      id: 268,
      resource_uri: "http://localhost:8000/v1/cities/268/",
      name: "Serpuhov"
    }


### Permissions

Extensions for [permissions](http://www.django-rest-framework.org/api-guide/permissions.html).

#### Object permissions

*New in DRF-extensions Development version*

Django Rest Framework allows you to use [DjangoObjectPermissions](http://www.django-rest-framework.org/api-guide/permissions#djangoobjectpermissions) out of the box. But it has one limitation - if user has no permissions for viewing resource he will get `404` as response code. In most cases it's good approach because it solves security issues by default. But what if you wanted to return `401` or `403`? What if you wanted to say to user - "You need to be logged in for viewing current resource" or "You don't have permissions for viewing current resource"?

`ExtenedDjangoObjectPermissions` will help you to be more flexible. By default it behaves as standard [DjangoObjectPermissions](http://www.django-rest-framework.org/api-guide/permissions#djangoobjectpermissions). For example, it is safe to replace `DjangoObjectPermissions` with extended permissions class:

    from rest_framework_extensions.permissions import (
        ExtendedDjangoObjectPermissions as DjangoObjectPermissions
    )

    class CommentView(viewsets.ModelViewSet):
        permission_classes = (DjangoObjectPermissions,)

Now every request from unauthorized user will get `404` response:

    # Request
    GET /comments/1/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 404 NOT FOUND
    Content-Type: application/json; charset=UTF-8

    {"detail": "Not found"}

With `ExtenedDjangoObjectPermissions` you can disable hiding forbidden for read objects by changing `hide_forbidden_for_read_objects` attribute:

    from rest_framework_extensions.permissions import (
        ExtendedDjangoObjectPermissions
    )

    class CommentViewObjectPermissions(ExtendedDjangoObjectPermissions):
        hide_forbidden_for_read_objects = False

    class CommentView(viewsets.ModelViewSet):
        permission_classes = (CommentViewObjectPermissions,)

Now lets see request response for user that has no permissions for viewing `CommentView` object:

    # Request
    GET /comments/1/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 403 FORBIDDEN
    Content-Type: application/json; charset=UTF-8

    {u'detail': u'You do not have permission to perform this action.'}

`ExtenedDjangoObjectPermissions` could be used only with Django Rest Framework >= 2.3.8, because [DjangoObjectPermissions](http://www.django-rest-framework.org/topics/release-notes#238) was introduced in 2.3.8 version.


### Caching

To cache something is to save the result of an expensive calculation so that you don’t have to perform the calculation next time. Here’s some pseudocode explaining how this would work for a dynamically generated api response:

    given a URL, try finding that API response in the cache
    if the response is in the cache:
        return the cached response
    else:
        generate the response
        save the generated response in the cache (for next time)
        return the generated response

#### Cache response

DRF-extensions allows you to cache api responses with simple `@cache_response` decorator.
There are two requirements for decorated method:

* It should be method of class which is inherited from `rest_framework.views.APIView`
* It should return `rest_framework.response.Response` instance.

Usage example:

    from rest_framework.response import Response
    from rest_framework import views
    from rest_framework_extensions.cache.decorators import (
        cache_response
    )
    from myapp.models import City

    class CityView(views.APIView):
        @cache_response()
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)

If you request view first time you'll get it from processed SQL query. (~60ms response time):

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['Moscow', 'London', 'Paris']

Second request will hit the cache. No sql evaluation, no database query. (~30 ms response time):

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['Moscow', 'London', 'Paris']

Reduction in response time depends on calculation complexity inside your API method. Sometimes it reduces from 1 second to 10ms, sometimes you win just 10ms.

#### Timeout

You can specify cache timeout in seconds, providing first argument:

    class CityView(views.APIView):
        @cache_response(60 * 15)
        def get(self, request, *args, **kwargs):
            ...

In the above example, the result of the `get()` view will be cached for 15 minutes.

If you don't specify `cache_timout` argument then value from `REST_FRAMEWORK_EXTENSIONS` settings will be used. By default it's `None`, which means "cache forever". You can change this default in settings:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_CACHE_RESPONSE_TIMEOUT': 60 * 15
    }

#### Cache key

By default every cached data from `@cache_response` decorator stored by key, which calculated
with [DefaultKeyConstructor](#default-key-constructor).

You can change cache key by providing `key_func` argument, which must be callable:

    def calculate_cache_key(view_instance, view_method, 
                            request, args, kwargs):
        return '.'.join([
            len(args),
            len(kwargs)
        ])

    class CityView(views.APIView):
        @cache_response(60 * 15, key_func=calculate_cache_key)
        def get(self, request, *args, **kwargs):
            ...

You can implement view method and use it for cache key calculation by specifying `key_func` argument as string:

    class CityView(views.APIView):
        @cache_response(60 * 15, key_func='calculate_cache_key')
        def get(self, request, *args, **kwargs):
            ...

        def calculate_cache_key(self, view_instance, view_method,
                                request, args, kwargs):
            return '.'.join([
                len(args),
                len(kwargs)
            ])

Key calculation function will be called with next parameters:

* **view_instance** - view instance of decorated method
* **view_method** - decorated method
* **request** - decorated method request
* **args** - decorated method positional arguments
* **kwargs** - decorated method keyword arguments

#### Default key function

If `@cache_response` decorator used without key argument then default key function will be used. You can change this function in
settings:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_CACHE_KEY_FUNC':
          'rest_framework_extensions.utils.default_cache_key_func'
    }

`default_cache_key_func` uses [DefaultKeyConstructor](#default-key-constructor) as a base for key calculation.

#### CacheResponseMixin

It is common to cache standard [viewset](http://www.django-rest-framework.org/api-guide/viewsets) `retrieve` and `list`
methods. That is why `CacheResponseMixin` exists. Just mix it into viewset implementation and those methods will
use functions, defined in `REST_FRAMEWORK_EXTENSIONS` [settings](#settings):

* *"DEFAULT\_OBJECT\_CACHE\_KEY\_FUNC"* for `retrieve` method
* *"DEFAULT\_LIST\_CACHE\_KEY\_FUNC"* for `list` method

By default those functions are using [DefaultKeyConstructor](#default-key-constructor) and extends it:

* With `RetrieveSqlQueryKeyBit` for *"DEFAULT\_OBJECT\_CACHE\_KEY\_FUNC"*
* With `ListSqlQueryKeyBit` and `PaginationKeyBit` for *"DEFAULT\_LIST\_CACHE\_KEY\_FUNC"*

You can change those settings for custom cache key generation:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_OBJECT_CACHE_KEY_FUNC':
          'rest_framework_extensions.utils.default_object_cache_key_func',
        'DEFAULT_LIST_CACHE_KEY_FUNC':
          'rest_framework_extensions.utils.default_list_cache_key_func',
    }

Mixin example usage:

    from myapps.serializers import UserSerializer
    from rest_framework_extensions.cache.mixins import CacheResponseMixin

    class UserViewSet(CacheResponseMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer

You can change cache key function by providing `object_cache_key_func` or
`list_cache_key_func` methods in view class:

    class UserViewSet(CacheResponseMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer

        def object_cache_key_func(self, **kwargs):
            return 'some key for object'

        def list_cache_key_func(self, **kwargs):
            return 'some key for list'

Ofcourse you can use custom [key constructor](#key-constructor):

    from yourapp.key_constructors import (
        CustomObjectKeyConstructor,
        CustomListKeyConstructor,
    )

    class UserViewSet(CacheResponseMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer
        object_cache_key_func = CustomObjectKeyConstructor()
        list_cache_key_func = CustomListKeyConstructor()

If you want to cache only `retrieve` method then you could use `rest_framework_extensions.cache.mixins.RetrieveCacheResponseMixin`.

If you want to cache only `list` method then you could use `rest_framework_extensions.cache.mixins.ListCacheResponseMixin`.


### Key constructor

As you could see from previous section cache key calculation might seem fairly simple operation. But let's see next example. We make ordinary HTTP request to cities resource:

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['Moscow', 'London', 'Paris']

By the moment all goes fine - response returned and cached. Let's make the same request requiring XML response:

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/xml

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    ['Moscow', 'London', 'Paris']

What is that? Oh, we forgot about format negotiations. We can add format to key bits:

    def calculate_cache_key(view_instance, view_method, 
                            request, args, kwargs):
        return '.'.join([
            len(args),
            len(kwargs),
            request.accepted_renderer.format  # here it is
        ])

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/xml

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/xml; charset=UTF-8

    <?xml version="1.0" encoding="utf-8"?>
    <root>
        <list-item>Moscow</list-item>
        <list-item>London</list-item>
        <list-item>Paris</list-item>
    </root>

That's cool now - we have different responses for different formats with different cache keys. But there are many cases, where key should be different for different requests:

* Response format (json, xml);
* User (exact authorized user or anonymous);
* Different request meta data (request.META['REMOTE_ADDR']);
* Language (ru, en);
* Headers;
* Query params. For example, `jsonp` resources need `callback` param, which rendered in response;
* Pagination. We should show different data for different pages;
* Etc...

Ofcourse we can use custom `calculate_cache_key` methods and reuse them for different API methods, but we can't reuse just parts of them. For example, one method depends on user id and language, but another only on user id. How to be more DRYish? Let's see some magic:

    from rest_framework_extensions.key_constructor.constructors import (
        KeyConstructor
    )
    from rest_framework_extensions.key_constructor import bits
    from your_app.utils import get_city_by_ip

    class CityGetKeyConstructor(KeyConstructor):
        unique_method_id = bits.UniqueMethodIdKeyBit()
        format = bits.FormatKeyBit()
        language = bits.LanguageKeyBit()

    class CityHeadKeyConstructor(CityGetKeyConstructor):
        user = bits.UserKeyBit()
        request_meta = bits.RequestMetaKeyBit(params=['REMOTE_ADDR'])

    class CityView(views.APIView):
        @cache_response(key_func=CityGetKeyConstructor())
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)

        @cache_response(key_func=CityHeadKeyConstructor())
        def head(self, request, *args, **kwargs):
            city = ''
            user = self.request.user
            if user.is_authenticated() and user.city:
                city = Response(user.city.name)
            if not city:
                city = get_city_by_ip(request.META['REMOTE_ADDR'])
            return Response(city)

Firstly, let's revise `CityView.get` method cache key calculation. It constructs from 3 bits:

* **unique\_method\_id** - remember that [default key calculation](#cache-key)? Here it is. Just one of the cache key bits. `head` method has different set of bits and they can't collide with `get` method bits. But there could be another view class with the same bits.
* **format** - key would be different for different formats.
* **language** - key would be different for different languages.

The second method `head` has the same `unique_method_id`, `format` and `language` bits, buts extends with 2 more:

* **user** - key would be different for different users. As you can see in response calculation we use `request.user` instance. For different users we need different responses.
* **request_meta** - key would be different for different ip addresses. As you can see in response calculation we are falling back to getting city from ip address if couldn't get it from authorized user model.

#### Default key bits

Out of the box DRF-extensions has some basic key bits. They are all located in `rest_framework_extensions.key_constructor.bits` module.

**FormatKeyBit**

Retrieves format info from request. Usage example:

    class MyKeyConstructor(KeyConstructor):
        format = FormatKeyBit()

**LanguageKeyBit**

Retrieves active language for request. Usage example:

    class MyKeyConstructor(KeyConstructor):
        user = LanguageKeyBit()

**UserKeyBit**

Retrieves user id from request. If it is anonymous then returnes *"anonymous"* string. Usage example:

    class MyKeyConstructor(KeyConstructor):
        user = UserKeyBit()

**RequestMetaKeyBit**

Retrieves data from [request.META](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.HttpRequest.META) dict.
Usage example:

    class MyKeyConstructor(KeyConstructor):
        ip_address_and_user_agent = RequestMetaKeyBit(
            ['REMOTE_ADDR', 'HTTP_USER_AGENT']
        )

**HeadersKeyBit**

Same as `RequestMetaKeyBit` retrieves data from [request.META](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.HttpRequest.META) dict.
The difference is that `HeadersKeyBit` allows to use normal header names:

    class MyKeyConstructor(KeyConstructor):
        user_agent_and_geobase_id = HeadersKeyBit(
            ['user-agent', 'x-geobase-id']
        )
        # will process request.META['HTTP_USER_AGENT'] and
        #              request.META['HTTP_X_GEOBASE_ID']

**QueryParamsKeyBit**

Retrieves data from [request.GET](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.HttpRequest.GET) dict.
Usage example:

    class MyKeyConstructor(KeyConstructor):
        part_and_callback = bits.QueryParamsKeyBit(
            ['part', 'callback']
        )

**PaginationKeyBit**

Inherits from `QueryParamsKeyBit` and returns data from used pagination params.

    class MyKeyConstructor(KeyConstructor):
        pagination = bits.PaginationKeyBit()

**ListSqlQueryKeyBit**

Retrieves sql query for `view.filter_queryset(view.get_queryset())` filtering.

    class MyKeyConstructor(KeyConstructor):
        list_sql_query = bits.ListSqlQueryKeyBit()

**RetrieveSqlQueryKeyBit**

Retrieves sql query for retrieving exact object.

    class MyKeyConstructor(KeyConstructor):
        retrieve_sql_query = bits.RetrieveSqlQueryKeyBit()

**UniqueViewIdKeyBit**

Combines data about view module and view class name.

    class MyKeyConstructor(KeyConstructor):
        unique_view_id = bits.UniqueViewIdKeyBit()

**UniqueMethodIdKeyBit**

Combines data about view module, view class name and view method name.

    class MyKeyConstructor(KeyConstructor):
        unique_view_id = bits.UniqueMethodIdKeyBit()


#### Default key constructor

`DefaultKeyConstructor` is located in `rest_framework_extensions.key_constructor.constructors` module and constructs a key
from unique method id, request format and request language. It has next implementation:

    class DefaultKeyConstructor(KeyConstructor):
        unique_method_id = bits.UniqueMethodIdKeyBit()
        format = bits.FormatKeyBit()
        language = bits.LanguageKeyBit()


#### How key constructor works

Key constructor class works in the same manner as the standard [django forms](https://docs.djangoproject.com/en/dev/topics/forms/) and
key bits used like form fields. Lets go through key construction steps for [DefaultKeyConstructor](#default-key-constructor).

Firstly, constructor starts iteration over every key bit:

* **unique\_method\_id**
* **format**
* **language**

Then constructor gets data from every key bit calling method `get_data`:

* **unique\_method\_id** - `u'your_app.views.SometView.get'`
* **format** - `u'json'`
* **language** - `u'en'`

Every key bit `get_data` method is called with next arguments:

* **view_instance** - view instance of decorated method
* **view_method** - decorated method
* **request** - decorated method request
* **args** - decorated method positional arguments
* **kwargs** - decorated method keyword arguments

After this it combines every key bit data to one dict, which keys are a key bits names in constructor, and values are returned data:

    {
        'unique_method_id': u'your_app.views.SometView.get',
        'format': u'json',
        'language': u'en'
    }

Then constructor dumps resulting dict to json:

    '{"unique_method_id": "your_app.views.SometView.get", "language": "en", "format": "json"}'

And finally compresses json with **sha256** and returns hash value:

    'b04f8f03c89df824e0ecd25230a90f0e0ebe184cf8c0114342e9471dd2275baa'

#### Custom key bit

We are going to create a simple key bit which could be used in real applications with next properties:

* High read rate
* Low write rate

The task is - cache every read request and invalidate all cache data after write to any model, which used in API. This approach
let us don't think about granular cache invalidation - just flush it after any model instance change/creation/deletion.

Lets create models:

    # models.py
    from django.db import models

    class Group(models.Model):
        title = models.CharField()

    class Profile(models.Model):
        name = models.CharField()
        group = models.ForeignKey(Group)

Define serializers:

    # serializers.py
    from yourapp.models import Group, Profile
    from rest_framework import serializers

    class GroupSerializer(serializers.ModelSerializer):
        class Meta:
            model = Group

    class ProfileSerializer(serializers.ModelSerializer):
        group = GroupSerializer()

        class Meta:
            model = Profile

Create views:

    # views.py
    from yourapp.serializers import GroupSerializer, ProfileSerializer
    from yourapp.models import Group, Profile

    class GroupViewSet(viewsets.ReadOnlyModelViewSet):
        serializer_class = GroupSerializer
        queryset = Group.objects.all()

    class ProfileViewSet(viewsets.ReadOnlyModelViewSet):
        serializer_class = ProfileSerializer
        queryset = Profile.objects.all()

And finally register views in router:

    # urls.py
    from yourapp.views import GroupViewSet,ProfileViewSet

    router = DefaultRouter()
    router.register(r'groups', GroupViewSet)
    router.register(r'profiles', ProfileViewSet)
    urlpatterns = router.urls

At the moment we have API, but it's not cached. Lets cache it and create our custom key bit:

    # views.py
    import datetime
    from django.core.cache import cache
    from django.utils.encoding import force_text
    from yourapp.serializers import GroupSerializer, ProfileSerializer
    from rest_framework_extensions.cache.decorators import cache_response
    from rest_framework_extensions.key_constructor.constructors import (
        DefaultKeyConstructor
    )
    from rest_framework_extensions.key_constructor.bits import (
        KeyBitBase,
        RetrieveSqlQueryKeyBit,
        ListSqlQueryKeyBit,
        PaginationKeyBit
    )

    class UpdatedAtKeyBit(KeyBitBase):
        def get_data(self, **kwargs):
            key = 'api_update_at_timestamp'
            value = cache.get('api_updated_at_timestamp', None)
            if not value:
                value = datetime.datetime.utcnow()
                cache.set(key, value=value)
            return force_text(value)

    class CustomObjectKeyConstructor(DefaultKeyConstructor):
        retrieve_sql = RetrieveSqlQueryKeyBit()
        updated_at = UpdatedAtKeyBit()

    class CustomListKeyConstructor(DefaultKeyConstructor):
        list_sql = ListSqlQueryKeyBit()
        pagination = PaginationKeyBit()
        updated_at = UpdatedAtKeyBit()

    class GroupViewSet(viewsets.ReadOnlyModelViewSet):
        serializer_class = GroupSerializer

        @cache_response(key_func=CustomObjectKeyConstructor())
        def retrieve(self, *args, **kwargs):
            return super(GroupViewSet, self).retrieve(*args, **kwargs)

        @cache_response(key_func=CustomListKeyConstructor())
        def list(self, *args, **kwargs):
            return super(GroupViewSet, self).list(*args, **kwargs)

    class ProfileViewSet(viewsets.ReadOnlyModelViewSet):
        serializer_class = ProfileSerializer

        @cache_response(key_func=CustomObjectKeyConstructor())
        def retrieve(self, *args, **kwargs):
            return super(ProfileViewSet, self).retrieve(*args, **kwargs)

        @cache_response(key_func=CustomListKeyConstructor())
        def list(self, *args, **kwargs):
            return super(ProfileViewSet, self).list(*args, **kwargs)

As you can see `UpdatedAtKeyBit` just adds to key information when API models has been update last time. If there is no
information about it then new datetime will be used for key bit data.

Lets write cache invalidation. We just connect models to standard signals and change value in cache by key `api_updated_at_timestamp`:

    # models.py
    import datetime
    from django.db import models
    from django.db.models.signals import post_save, post_delete

    def change_api_updated_at(sender=None, instance=None, *args, **kwargs):
        cache.set('api_updated_at_timestamp', datetime.datetime.utcnow())

    class Group(models.Model):
        title = models.CharField()

    class Profile(models.Model):
        name = models.CharField()
        group = models.ForeignKey(Group)

    for model in [Group, Profile]:
        post_save.connect(receiver=change_api_updated_at, sender=model)
        post_delete.connect(receiver=change_api_updated_at, sender=model)

And that's it. When any model changes then value in cache by key `api_updated_at_timestamp` will be changed too. After this every
key constructor, that used `UpdatedAtKeyBit`, will construct new keys and `@cache_response` decorator will
cache data in new places.


### Conditional requests

*This documentation section uses information from [RESTful Web Services Cookbook](http://shop.oreilly.com/product/9780596801694.do) 10-th chapter.*

Conditional HTTP request allows API clients to accomplish 2 goals:

* Conditional HTTP GET saves client and server time and bandwidth.
* For unsafe requests such as PUT, POST, and DELETE, conditional requests provide concurrency control.

#### HTTP Etag

*An ETag or entity tag, is part of HTTP, the protocol for the World Wide Web. It is one of several mechanisms that HTTP provides for web cache validation, and which allows a client to make conditional requests.* - [Wikipedia](http://en.wikipedia.org/wiki/HTTP_ETag)

For etag calculation and conditional request processing you should use `rest_framework_extensions.etag.decorators.etag` decorator. It's similar to native [django decorator](https://docs.djangoproject.com/en/dev/topics/conditional-view-processing/).

    from rest_framework_extensions.etag.decorators import etag

    class CityView(views.APIView):
        @etag()
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)

By default `@etag` would calculate header value with the same algorithm as [cache key](#cache-key) default calculation performs.

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    ETag: "e7b50490dc546d116635a14cfa58110306dd6c5434146b6740ec08bf0a78f9a2"

    ['Moscow', 'London', 'Paris']

You can define custom function for Etag value calculation with `etag_func` argument:

    from rest_framework_extensions.etag.decorators import etag

    def calculate_etag(view_instance, view_method, 
                       request, args, kwargs):
        return '.'.join([
            len(args),
            len(kwargs)
        ])

    class CityView(views.APIView):
        @etag(etag_func=calculate_etag)
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)


You can implement view method and use it for Etag calculation by specifying `etag_func` argument as string:

    from rest_framework_extensions.etag.decorators import etag

    class CityView(views.APIView):
        @etag(etag_func='calculate_etag_from_method')
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)

        def calculate_etag_from_method(self, view_instance, view_method,
                                       request, args, kwargs):
            return '.'.join([
                len(args),
                len(kwargs)
            ])

Etag calculation function will be called with next parameters:

* **view_instance** - view instance of decorated method
* **view_method** - decorated method
* **request** - decorated method request
* **args** - decorated method positional arguments
* **kwargs** - decorated method keyword arguments

#### Default etag function

If `@etag` decorator used without `etag_func` argument then default etag function will be used. You can change this function in
settings:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_ETAG_FUNC':
          'rest_framework_extensions.utils.default_etag_func'
    }

`default_etag_func` uses [DefaultKeyConstructor](#default-key-constructor) as a base for etag calculation.

#### Usage with caching

As you can see `@etag` and `@cache_response` decorators has similar key calculation approaches. They both can take key from simple callable function. And more then this - in many cases they share the same calculation logic. In the next example we use both decorators, which share one calculation function:

    from rest_framework_extensions.etag.decorators import etag
    from rest_framework_extensions.cache.decorators import cache_response
    from rest_framework_extensions.key_constructor import bits
    from rest_framework_extensions.key_constructor.constructors import (
        KeyConstructor
    )

    class CityGetKeyConstructor(KeyConstructor):
        format = bits.FormatKeyBit()
        language = bits.LanguageKeyBit()

    class CityView(views.APIView):
        key_constructor_func = CityGetKeyConstructor()

        @etag(key_constructor_func)
        @cache_response(key_func=key_constructor_func)
        def get(self, request, *args, **kwargs):
            cities = City.objects.all().values_list('name', flat=True)
            return Response(cities)

Note the decorators order. First goes `@etag` and after goes `@cache_response`. We want firstly perform conditional processing and after it response processing.

There is one more point for it. If conditional processing didn't fail then `key_constructor_func` would be called again in `@cache_response`.
But in most cases first calculation is enough. To accomplish this goal you could use `KeyConstructor` initial argument `memoize_for_request`:

    >>> key_constructor_func = CityGetKeyConstructor(memoize_for_request=True)
    >>> request1, request1 = 'request1', 'request2'
    >>> print key_constructor_func(request=request1)  # full calculation
    request1-key
    >>> print key_constructor_func(request=request1)  # data from cache
    request1-key
    >>> print key_constructor_func(request=request2)  # full calculation
    request2-key
    >>> print key_constructor_func(request=request2)  # data from cache
    request2-key

By default `memoize_for_request` is `False`, but you can change it in settings:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_KEY_CONSTRUCTOR_MEMOIZE_FOR_REQUEST': True
    }


It's important to note that this memoization is thread safe.


#### Saving time and bandwith

When a server returns `ETag` header, you should store it along with the representation data on the client.
When making GET and HEAD requests for the same resource in the future, include the `If-None-Match` header
to make these requests "conditional".

For example, retrieve all cities:

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    ETag: "some_etag_value"

    ['Moscow', 'London', 'Paris']

If you make same request with `If-None-Match` and there is the cached value for this request,
then server will respond with `304` status code without body data.

    # Request
    GET /cities/ HTTP/1.1
    Accept: application/json
    If-None-Match: some_etag_value

    # Response
    HTTP/1.1 304 NOT MODIFIED
    Content-Type: application/json; charset=UTF-8
    Etag: "some_etag_value"

After this response you can use existing cities data on the client.

#### Concurrency control

Concurrency control ensures the correct processing of data under concurrent operations by clients.
There are two ways to implement concurrency control:

* **Pessimistic concurrency control**. In this model, the client gets a lock, obtains
the current state of the resource, makes modifications, and then releases the lock.
During this process, the server prevents other clients from acquiring a lock on the same resource.
Relational databases operate in this manner.
* **Optimistic concurrency control**. In this model, the client first gets a token.
Instead of obtaining a lock, the client attempts a write operation with the token included in the request.
The operation succeeds if the token is still valid and fails otherwise.

HTTP, being a stateless application control, is designed for optimistic concurrency control.

                                    PUT
                                     |
                              +-------------+
                              |  Etag       |
                              |  supplied?  |
                              +-------------+
                               |           |
                              Yes          No
                               |           |
            +--------------------+       +-----------------------+
            |  Do preconditions  |       |  Does the             |
            |  match?            |       |  resource exist?      |
            +--------------------+       +-----------------------+
                |           |                   |              |
                Yes         No                  Yes            No
                |           |                   |              |
    +--------------+  +--------------------+  +-------------+  |
    |  Update the  |  |  412 Precondition  |  |  403        |  |
    |  resource    |  |  failed            |  |  Forbidden  |  |
    +--------------+  +--------------------+  +-------------+  |
                                                               |
                                         +-----------------------+
                                         |  Can clients          |
                                         |  create resources     |
                                         +-----------------------+
                                               |           |
                                              Yes          No
                                               |           |
                                         +-----------+   +-------------+
                                         |  201      |   |  404        |
                                         |  Created  |   |  Not Found  |
                                         +-----------+   +-------------+

Delete:

                                   DELETE
                                     |
                              +-------------+
                              |  Etag       |
                              |  supplied?  |
                              +-------------+
                               |           |
                              Yes          No
                               |           |
            +--------------------+       +-------------+
            |  Do preconditions  |       |  403        |
            |  match?            |       |  Forbidden  |
            +--------------------+       +-------------+
                |           |
                Yes         No
                |           |
    +--------------+  +--------------------+
    |  Delete the  |  |  412 Precondition  |
    |  resource    |  |  failed            |
    +--------------+  +--------------------+


Here is example of implementation for all CRUD methods (except create, because it doesn't need concurrency control)
wrapped with `etag` decorator:

    from rest_framework.viewsets import ModelViewSet
    from rest_framework_extensions.key_constructor import bits
    from rest_framework_extensions.key_constructor.constructors import (
        KeyConstructor
    )

    from your_app.models import City
    from your_app.key_bits import UpdatedAtKeyBit

    class CityListKeyConstructor(KeyConstructor):
        format = bits.FormatKeyBit()
        language = bits.LanguageKeyBit()
        pagination = bits.PaginationKeyBit()
        list_sql_query = bits.ListSqlQueryKeyBit()
        unique_view_id = bits.UniqueViewIdKeyBit()

    class CityDetailKeyConstructor(KeyConstructor):
        format = bits.FormatKeyBit()
        language = bits.LanguageKeyBit()
        retrieve_sql_query = bits.RetrieveSqlQueryKeyBit()
        unique_view_id = bits.UniqueViewIdKeyBit()
        updated_at = UpdatedAtKeyBit()

    class CityViewSet(ModelViewSet):
        list_key_func = CityListKeyConstructor(
            memoize_for_request=True
        )
        obj_key_func = CityDetailKeyConstructor(
            memoize_for_request=True
        )

        @etag(list_key_func)
        @cache_response(key_func=list_key_func)
        def list(self, request, *args, **kwargs):
            return super(CityViewSet, self).list(request, *args, **kwargs)

        @etag(obj_key_func)
        @cache_response(key_func=obj_key_func)
        def retrieve(self, request, *args, **kwargs):
            return super(CityViewSet, self).retrieve(request, *args, **kwargs)

        @etag(obj_key_func)
        def update(self, request, *args, **kwargs):
            return super(CityViewSet, self).update(request, *args, **kwargs)

        @etag(obj_key_func)
        def destroy(self, request, *args, **kwargs):
            return super(CityViewSet, self).destroy(request, *args, **kwargs)


#### Etag for unsafe methods

From previous section you could see that unsafe methods, such `update` (PUT, PATCH) or `destroy` (DELETE), have the same `@etag`
decorator wrapping manner as the safe methods.

But every unsafe method has one distinction from safe method - it changes the data
which could be used for Etag calculation. In our case it is `UpdatedAtKeyBit`. It means that we should calculate Etag:

* Before building response - for `If-Match` and `If-None-Match` conditions validation
* After building response (if necessary) - for clients

`@etag` decorator has special attribute `rebuild_after_method_evaluation`, which by default is `False`.

If you specify `rebuild_after_method_evaluation` as `True` then Etag will be rebuilt after method evaluation:

    class CityViewSet(ModelViewSet):
        ...
        @etag(obj_key_func, rebuild_after_method_evaluation=True)
        def update(self, request, *args, **kwargs):
            return super(CityViewSet, self).update(request, *args, **kwargs)

        @etag(obj_key_func)
        def destroy(self, request, *args, **kwargs):
            return super(CityViewSet, self).destroy(request, *args, **kwargs)

    # Request
    PUT /cities/1/ HTTP/1.1
    Accept: application/json

    {"name": "London"}

    # Response
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    ETag: "4e63ef056f47270272b96523f51ad938b5ea141024b767880eac047d10a0b339"

    {
      id: 1,
      name: "London"
    }

As you can see we didn't specify `rebuild_after_method_evaluation` for `destroy` method. That is because there is no
sense to use returned Etag value on clients if object deletion already performed.

With `rebuild_after_method_evaluation` parameter Etag calculation for `PUT`/`PATCH` method would look like:

                 +--------------+
                 |    Request   |
                 +--------------+
                        |
           +--------------------------+
           |  Calculate Etag          |
           |  for condition matching  |
           +--------------------------+
                        |
              +--------------------+
              |  Do preconditions  |
              |  match?            |
              +--------------------+
                  |           |
                  Yes         No
                  |           |
      +--------------+  +--------------------+
      |  Update the  |  |  412 Precondition  |
      |  resource    |  |  failed            |
      +--------------+  +--------------------+
             |
    +--------------------+
    |  Calculate Etag    |
    |  again and add it  |
    |  to response       |
    +--------------------+
             |
       +------------+
       |  Return    |
       |  response  |
       +------------+

`If-None-Match` example for `DELETE` method:

    # Request
    DELETE /cities/1/ HTTP/1.1
    Accept: application/json
    If-None-Match: some_etag_value

    # Response
    HTTP/1.1 304 NOT MODIFIED
    Content-Type: application/json; charset=UTF-8
    Etag: "some_etag_value"


`If-Match` example for `DELETE` method:

    # Request
    DELETE /cities/1/ HTTP/1.1
    Accept: application/json
    If-Match: another_etag_value

    # Response
    HTTP/1.1 412 PRECONDITION FAILED
    Content-Type: application/json; charset=UTF-8
    Etag: "some_etag_value"


#### ETAGMixin

It is common to process etags for standard [viewset](http://www.django-rest-framework.org/api-guide/viewsets)
`retrieve`, `list`, `update` and `destroy` methods.
That is why `ETAGMixin` exists. Just mix it into viewset
implementation and those methods will use functions, defined in `REST_FRAMEWORK_EXTENSIONS` [settings](#settings):

* *"DEFAULT\_OBJECT\_ETAG\_FUNC"* for `retrieve`, `update` and `destroy` methods
* *"DEFAULT\_LIST\_ETAG\_FUNC"* for `list` method

By default those functions are using [DefaultKeyConstructor](#default-key-constructor) and extends it:

* With `RetrieveSqlQueryKeyBit` for *"DEFAULT\_OBJECT\_ETAG\_FUNC"*
* With `ListSqlQueryKeyBit` and `PaginationKeyBit` for *"DEFAULT\_LIST\_ETAG\_FUNC"*

You can change those settings for custom etag generation:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_OBJECT_ETAG_FUNC':
          'rest_framework_extensions.utils.default_object_etag_func',
        'DEFAULT_LIST_ETAG_FUNC':
          'rest_framework_extensions.utils.default_list_etag_func',
    }

Mixin example usage:

    from myapps.serializers import UserSerializer
    from rest_framework_extensions.etag.mixins import ETAGMixin

    class UserViewSet(ETAGMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer

You can change etag function by providing `object_etag_func` or
`list_etag_func` methods in view class:

    class UserViewSet(ETAGMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer

        def object_etag_func(self, **kwargs):
            return 'some key for object'

        def list_etag_func(self, **kwargs):
            return 'some key for list'

Ofcourse you can use custom [key constructor](#key-constructor):

    from yourapp.key_constructors import (
        CustomObjectKeyConstructor,
        CustomListKeyConstructor,
    )

    class UserViewSet(ETAGMixin, viewsets.ModelViewSet):
        serializer_class = UserSerializer
        object_etag_func = CustomObjectKeyConstructor()
        list_etag_func = CustomListKeyConstructor()

It is important to note that etags for unsafe method `update` is processed with parameter
`rebuild_after_method_evaluation` equals `True`. You can read why from [this](#etag-for-unsafe-methods) section.

There are other mixins for more granular Etag calculation in `rest_framework_extensions.etag.mixins` module:

* **ReadOnlyETAGMixin** - only for `retrieve` and `list` methods
* **RetrieveETAGMixin** - only for `retrieve` method
* **ListETAGMixin** - only for `list` method
* **DestroyETAGMixin** - only for `destroy` method
* **UpdateETAGMixin** - only for `update` method


### Settings

DRF-extesions follows Django Rest Framework approach in settings implementation.

[In Django Rest Framework](http://www.django-rest-framework.org/api-guide/settings) you specify custom settings by changing `REST_FRAMEWORK` variable in settings file:

    REST_FRAMEWORK = {
        'DEFAULT_RENDERER_CLASSES': (
            'rest_framework.renderers.YAMLRenderer',
        ),
        'DEFAULT_PARSER_CLASSES': (
            'rest_framework.parsers.YAMLParser',
        )
    }

In DRF-extesions there is a magic variable too called `REST_FRAMEWORK_EXTENSIONS`:

    REST_FRAMEWORK_EXTENSIONS = {
        'DEFAULT_CACHE_RESPONSE_TIMEOUT': 60 * 15
    }

#### Accessing settings

If you need to access the values of DRF-exteinsions API settings in your project, you should use the `extensions_api_settings` object. For example:

    from rest_framework_extensions.settings import extensions_api_settings

    print extensions_api_settings.DEFAULT_CACHE_RESPONSE_TIMEOUT


### Release notes

You can read about versioning, deprecation policy and upgrading from
[Django REST framework documentation](http://django-rest-framework.org/topics/release-notes).

#### New in development version

* Added [PaginateByMaxMixin](#paginatebymaxmixin)
* Added [ExtenedDjangoObjectPermissions](#object-permissions)

#### 0.2.1

*Feb. 1, 2014*

* Rewritten tests to nose and tox
* New tests directory structure
* Rewritten HTTP documentation requests examples into more raw manner
* Added trailing_slash on extended routers for Django Rest Framework versions`>=2.3.6` (which supports this feature)
* Added [caching](#caching)
* Added [key constructor](#key-constructor)
* Added [conditional requests](#conditional-requests) with Etag calculation
* Added [Cache/ETAG mixins](#cache-etag-mixins)
* Added [CacheResponseMixin](#cacheresponsemixin)
* Added [ETAGMixin](#etagmixin)
* Documented [ResourceUriField](#resourceurifield)
* Documented [settings](#settings) customization

#### 0.2

*Nov. 5, 2013*

* Moved docs from readme to github pages
* Docs generation with [Backdoc](https://github.com/chibisov/backdoc)