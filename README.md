# REST FRAMEWORK QUICKSTART

## Set up

```bash
# Create the project directory
mkdir tutorial
cd tutorial

# Create a virtual environment to isolate our package dependencies locally
python3 -m venv env
./env\Scripts\activate

# Install Django and Django REST framework into the virtual environment
pip install djangorestframework

# Set up a new project with a single application
django-admin startproject rest_tutorial 
cd rest_tutorial
django-admin startapp quickstart
cd ..
```

* **所有的操作在虚拟环境下进行**

* 建立之后应该在第一时间**同步**

  `python manage.py migrate`

* 创建一个超管

  `python manage.py createsuperuser --username admin --email admin@example.com`

  密码是admin

## Serializers

* 首先需要定义的是一些序列化器作为数据的代表

* 创建一个新的的模块`quickstart/serializers.py`

  ```python
  from django.contrib.auth.models import User, Group
  from rest_framework import serializers
  
  
  
  class UerSerializer(serializers.HyperlinkedModelSerializer):
      class Meta:
          model = User
          fields = ['url', 'username', 'email', 'groups']
  
  
  
  class GroupSerializer(serializers.HyperlinkedModelSerializer):
      class Meta:
          model = Group
          fields = ['url', 'name']
  ```

* 这里使用了超链接的序列化方式，也可以使用主键等方式，但是**超链接是更REST化的做法**

## Views

* 在`quickstart/views.py`中添加相关的视图

  ```python
  from django.contrib.auth.models import Group, User
  from rest_framework import permissions, viewsets
  
  from quickstart.serializers import GroupSerializer, UerSerializer
  
  
  
  class UserViewSet(viewsets.ModelViewSet):
      '''
      API endpoint that allows users to be viewed or edited
      '''
      queryset = User.objects.all().order_by('-date_joined')
      serializer_class = UerSerializer
      permission_classes = [permissions.IsAuthenticated]
  
  
  
  class GroupViewSet(viewsets.ModelViewSet):
      '''
      API endpoint that allows groups to be viewed or edited
      '''
      queryset = Group.objects.all().order_by('name')
      serializer_class = GroupSerializer
      permission_classes = [permissions.IsAuthenticated]
  ```

* ViewSets整合了许多常用的视图方法

  * 可以将其打碎并逐一定义
  * 但是**使用ViewSets**通常是更合理且有条理的明智做法

## URLs

* 在`rest_tutorial/urls.py`中添加API的URL

  ```python
  from django.urls import include, path
  from rest_framework import routers
  
  from quickstart import views
  
  router = routers.DefaultRouter
  router.register(r'users', views.UserViewSet)
  router.register(r'group', views.GroupViewSet)
  
  # Wire up our API using automatic URL routing
  # Additionally, we include login URLs for the browsable API
  urlpatterns = [
      path('', include(router.urls)),
      path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
  ]
  ```

  * **由于使用了ViewSets而不是View，可以自动地生成URL配置，只需要注册路由器即可**
  * 同样可以使用原有的方法或者类的途径来定义视图和URL，并显式地配置
  * **如果网页需要提供登录等方法，通常使用API提供的登录路径是更合理的做法**

## Pagination

* 在`settings.py`中确定每一页返回多少的对象

  ```python
  REST_FRAMEWORK = {
      'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
      'PAGE_SIZE': 10
  }
  ```

## Settings

* 在`settings.py`中的`INSTALLED_APPS`内添加`'rest_framework'`

  ```python
  INSTALLED_APPS = [
      # ...
      'rest_framework',
  ]
  ```

## Testing our API

* 运行服务器`python manage.py runserver`
* 使用`curl`、`httpie`等方式访问开发服务器
  * 此时的访问路径是`http://127.0.0.1:8000/users/`

# Serialization

## Introduction

* 创建app后在在`settings.py`中标记下载

  ```bash
  pip pygments
  python manage.py startapp snippets
  ```

  ```python
  INSTALLED_APPS = [
      ...
      'rest_framework',
      'snippets',
  ]
  ```

  ### 创建一个Model来操作

  ```python
  class Snippet(models.Model):
      created = models.DateTimeField(auto_now_add=True)
      title = models.CharField(max_length=100, blank=True, default='')
      code = models.TextField()
      linenos = models.BooleanField(default=False)
      language = models.CharField(choices=LANGUAGE_CHOICES, default='pyton', max_length=100)
      style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)
  
      
      class Meta:
          ordering = ['created']
  ```

* **更改了models之后需要`migrate`**

### 创建一个Serializer类

* 建立WebAPI的第一件事是提供序列化和反序列化snippet实例成json等代表的方式

* 可以声明serializers来类似于表单地进行

  ```python
  from rest_framework import serializers
  from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES
  
  
  
  class SnippetSerializer(serializers.Serializer):
      id = serializers.IntegerField(read_only = True)
      title = serializers.CharField(required=False, allow_blank=True, max_length=100)
      code = serializers.CharField(style={'base_template': 'textarea.html'})
      linenos = serializers.BooleanField(required=False)
      language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
      style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')
  
      
      def create(self, validated_data):
          '''
          Create and return a new `Snippet` instane, given the validated data.
          '''
          return Snippet.objects.create(**validated_data)
      
  
      def update(self, instance, validated_data):
          '''
          Update and return an existing `Snippet` instance, given the validated data.
          '''
          instance.title = validated_data.get('title', instance.title)
          instance.code = validated_data.get('code', instance.code)
          instance.linenos = validated_data.get('linenos', instance.linenos)
          instance.language = validated_data.get('language', instance.language)
          instance.style = validated_data.get('style', instance.style)
          instance.save()
          return instance
  ```

  * 第一个部分指定了需要序列化的field
  * `update`和`create`方法定义了怎么创建或修改一个合法的实例
    * 这些修改和创建会在调用`serializer.save()`时被使用
  * serializer和Form很像，都有相近的合法化标记和多个属性
    * 有的filed flag规定了渲染的模板
      * 如`{'base_template': 'texarea.html'}`
      * 这和Form中的`widget=widgets.Textarea`等价
    * 同样也提供了ModelSerializer类用于对Model直接建立serializer

### 操作Serializers

* 进入django的shell

  ```shell
  python manage.py shell
  ```

  * 以下为django shell的内容

  ```python
  from snippets.models import Snippet
  from snippets.serializers import SnippetSerializer
  from rest_framework.renderers import JSONRenderer
  from rest_framework.parsers import JSONParser
  
  snippet = Snippet(code='foo = "bar"\n')
  snippet.save()
  
  snippet = Snippet(code='print("hello, world")\n')
  snippet.save()
  ```

  ```shell
  serializer = SnippetSerializer(snippet)
  serializer.data
  # {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}
  ```

  * 此时出现serializer的相关属性，即把一个Model实例转化成了Python本地数据类型


  ```shell
  content = JSONRenderer().render(serializer.data)
  content
  # b'{"id":2,"title":"","code":"print(\\"hello, world\\")\\n","linenos":false,"language":"python","style":"friendly"}'
  ```

  * 再用JSONRenderer将其渲染为json

    ```python
    import io
    
    stream = io.BytesIO(content)
    data = JSONParser().parse(stream)
    ```

  * 同样可以用io流和`JSONParser`解出json文件，用其创建实例

    ```python
    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # {'title': '', 'code': 'print("hello, world")', 'linenos': False, 'language': 'python', 'style': 'friendly'}
    serializer.save()
    # <Snippet: Snippet object>
    ```

  * 可以序列化`querysets`而不是是实例

    ```python
    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [{'id': 1, 'title': '', 'code': 'foo = "bar"\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}, {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}, {'id': 3, 'title': '', 'code': 'print("hello, world")', 'linenos': False, 'language': 'python', 'style': 'friendly'}]
    ```

  ### 使用ModelSerializers

  * 和ModelForm类似，REST提供了直接对Model进行序列化的类

    ```python
    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ['id', 'title', 'code', 'linenos', 'language', 'style']
    ```

  * `ModelSerializer`和自动设定field（指定的），以及实现默认的`create`和`update`方法

## 使用Serializer写View

* import

  ```python
  from django.http import HttpResponse, JsonResponse
  from django.views.decorators.csrf import csrf_exempt
  from rest_framework.parsers import JSONParser
  from snippets.models import Snippet
  from snippets.serializers import SnippetSerialize
  ```

  ```python
  @csrf_exempt
  def snippet_list(request):
      '''
      List all code snippets, or create a new snippet.
      '''
      if request.method == 'GET':
          snippets = Snippet.objects.all()
          serializer = SnippetSerializer(snippets, many=True)
          return JsonResponse(serializer.data, safe=False)
      
      elif request.method == 'POST':
          data = JSONParser().parse(request)
          serializer = SnippetSerializer(data=data)
          if serializer.is_valid():
              serializer.save()
              return JsonResponse(serializer.data, status=201)
          return JsonResponse(serializer.errors, status=400)
  ```

  * 一个简单的view，创建或者展示snippet

    * 这里的`csrf_exempt`并不是一个合理的做法
    * 简单的去除了csrf保护

    ```python
    @csrf_exempt
    def snippet_detail(request, pk):
        '''
        Retrieve, update or delete a code snippet.
        '''
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)
        
        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JsonResponse(serializer.data)
        
        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JsonResponse(serializer.data)
            return JsonResponse(serializer.errors, status=400)
        
        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)
    ```

    * 这是一个根据请求类型执行对应操作的view

    * 一般需要这样的view来实现和Model的沟通

      ```python
      from django.urls import path
      from snippets import views
      
      urlpatterns = [
          path('snippets/', views.snippet_list),
          path('snippets/<int:pk>/', views.snippet_detail),
      ]
      ```

    * 在`snippets/urls.py`中添加页面url

      ```python
      urlpatterns = [
          path('', include('snippets.urls')),
      ]
      ```

    * 在`rest_tutorial/urls.py`中添加相关路由

    * 启动服务器，即可获取json回应

# Requests and Responses

## Request objects

* REST引入了`Request`对象
* 该对象继承了`HttpReqauest`
* 比较常用的功能是`request.POST`和`request.data`

## Response objects

* REST也引入了`Response`对象
* 使用`return Response(data)`来渲染并返回请求回应

## Status codes

* REST提供了一些显式的错误类型如`HTTP_400_BAD_REQUEST`
* 这些错误信息在`status`模块中
* 用其代替常见的http数字错误信息会更可读

## Wrapper

* REST提供了两种包装API views的方式
* `@api_view`修饰符（用于基于方法的视图方法）
* `APIView`类（用于基于类的视图方法）
* 这些包装类会提供`405 Method Not Allowed`等错误回应
* 并且处理获取`request.data`中可能出现的`ParseError`

## 重构Views

* 使用相关的类重新书写视图方法

  ```python
  from rest_framework import status
  from rest_framework.decorators import api_view
  from rest_framework.response import Response
  from snippets.models import Snippet
  from snippets.serializers import SnippetSerializer
  
  @api_view(['GET', 'POST'])
  def snippet_list(request):
      '''
      List all code snippets, or create a new snippet.
      '''
      if request.method == 'GET':
          snippets = Snippet.objects.all()
          serializer = SnippetSerializer(snippets, many=True)
          return Response(serializer.data)
      
      elif request.method == 'POST':
          serializer = SnippetSerializer(data=request.data)
          if serializer.is_valid():
              serializer.save()
              return Response(serializer.data, status=status.HTTP_201_CREATED)
          return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
      
      
  @api_view(['GET', 'PUT', 'DELETE'])
  def snippet_detail(request, pk):
      '''
      Retrieve, update or delete a code snippet.
      '''
      try:
          snippet = Snippet.objects.get(pk=pk)
      except Snippet.DoesNotExist:
          return Response(status=status.HTTP_404_NOT_FOUND)
      
      
      if request.method == 'GET':
          serializer = SnippetSerializer(snippet)
          return Response(serializer.data)
      
  
      elif request.method == 'PUT':
          serializer = SnippetSerializer(snippet, data=request.data)
          if serializer.is_valid():
              serializer.save()
              return Response(serializer.data)
          return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
      
  
      elif request.method == 'DELETE':
          snippet.delete()
          return Response(status=status.HTTP_204_NO_CONTENT)
  ```

  * 和django的传统写法差别不大，只需要把前文提到的部分替换即可

  * 不再需要显式使用JSON，并且`request.data`也可以处理`json`以外的请求

  * 但是为了避免开放的格式造成问题，可以在参数里显式传入请求的类型格式

    ```python
    def snippet_detail(request, pk, format=None)
    ```

  * 并在`snippets/urls.py`中添加后缀

    ```python
    from rest_framework.urlpatterns import format_suffix_patterns
    urlpatterns = format_suffix_patterns(urlpatterns)
    ```

  * 此时可以在用http访问时加入`Accept`来指定类型

    ```bash
    http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
    http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
    ```

  * 或者直接加后缀来表明类型

    ```bash
    http http://127.0.0.1:8000/snippets.json  # JSON suffix
    http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
    ```

  * 也可以在POST里指定类型

    ```bash
    # POST using form data
    http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"
    
    {
      "id": 3,
      "title": "",
      "code": "print(123)",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }
    
    # POST using JSON
    http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"
    
    {
        "id": 4,
        "title": "",
        "code": "print(456)",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }
    ```

* 默认下，因为在浏览器中渲染，会返回一个HTML格式文件
* 可以或者应该改进API的浏览器界面，让其变得更容易使用

# Class-based Views

## 使用Class-based的格式了重写view

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status



class SnippetList(APIView):
    '''
    List all snippets, or create a new snippet.
    '''
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)
    
    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)



class SnippetDetail(APIView):
    '''
    Retrieve, update or delete a snippet instance
    '''
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404
        
    
    def get(self, request, pk, format=None):
        snippet = self.get_object(pk=pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)
    

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk=pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk=pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

* 在`urls.py`中改用类式的视图调用

  ```python
  from django.urls import path
  from snippets import views
  
  urlpatterns = [
      path('snippets/', views.SnippetList.as_view()),
      path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
  ]
  
  
  from rest_framework.urlpatterns import format_suffix_patterns
  urlpatterns = format_suffix_patterns(urlpatterns)
  ```

## 使用mixins

* 使用class-based的views的优势在于可以将视图分解为不同的部分

* 而往往这些部分由REST提供了提前的实现

  ```python
  # Use mixins
  from snippets.models import Snippet
  from snippets.serializers import SnippetSerializer
  from rest_framework import mixins
  from rest_framework import generics
  
  
  class SnippetList(mixins.ListModelMixin,
                    mixins.CreateModelMixin,
                    generics.GenericAPIView):
      queryset = Snippet.objects.all()
      serializer_class = SnippetSerializer
  
  
      def get(self, request, *args, **kwargs):
          return self.list(request, **args, **kwargs)
      
  
      def post(self, request, *args, **kwargs):
          return self.create(request, *args, **kwargs)
  
  
  
  class SnippetDetail(mixins.RetrieveModelMixin,
                      mixins.UpdateModelMixin,
                      mixins.DestroyModelMixin,
                      generics.GenericAPIView):
      queryset = Snippet.objects.all()
      serializer_class = SnippetSerializer
  
  
      def get(self, request, *args, **kwargs):
          return self.retrieve(request, *args, **kwargs)
      
  
      def put(self, request, *args, **kwargs):
          return self.update(request, *args, **kwargs)
      
      def delete(self, request, *args, **kwargs):
          return self.destroy(request, *args, **kwargs)
  ```

* mixins的基类提供了list、create、detroy等方法来进行操作

* 只需要定义**对应的请求**所触发的**方法**即可

## 使用generic class-based views

* REST还提供了`generics`中的相应常用类来再次简化

  ```python
  from snippets.models import Snippet
  from snippets.serializers import SnippetSerializer
  from rest_framework import generics
  
  
  class SnippetList(generics.ListAPIView):
      queryset = Snippet.objects.all()
      serialize_class = SnippetSerializer
  
  
  
  class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
      queryset = Snippet.objects.all()
      serializer_class = SnippetSerializer
  ```

* 此时仍旧可以使用`Accpet`来指定类型


# 认证和权限

## 为Model添加高亮等功能

```python
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight

# in the Snippet model
	def save(self, *args, **kwargs):
        '''
        Use the 'pygments' library to create a highlighted HTML
        representation of the code snippet.
        '''

        lexer = get_lexer_by_name(self.language)
        lineons = 'table' if self.linenos else False
        options = {'title': self.title} if self.title else {}
        formatter = HtmlFormatter(style=self.style, lineons=lineons, full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super().save(*args, **kwargs)
```

* 删除原有的数据库和迁移，重新添加

  ```bash
  rm -f db.sqlite3
  rm -r snippets/migrations
  python manage.py makemigrations snippets
  python manage.py migrate
  ```

* 创建用户

  ```bash
  python manage.py createsuperuser
  ```

* 在序列化器中添加User的序列化器

  ```python
  from django.contrib.auth.models import User
  
  
  class UserSerializer(serializers.ModelSerializer):
      snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())
  
  
      class Meta:
          model = User
          fields = ['id', 'username', 'snippets']
  ```

  * 在REST中，所有使用的Model一般都有一个相关的序列化器提供Model的Representation
  * 此处的`Snippet`和`User`的联系在`Snippet`中指定，直接使用`User`作为Meta不会包含该属性

* 添加视图方法

  ```python
  from django.contrib.auth.models import User
  from snippets.serializers import UserSerializer
  
  
  class UserList(generics.ListAPIView):
      queryset = User.objects.all()
      serializer_class = UserSerializer
  
  
  
  class UserDetial(generics.RetrieveAPIView):
      queryset = User.objects.all()
      serializer_class = UserSerializer
  ```

* 添加url到`snippets/urls.py`

  ```python
      path('users/', views.UserList.as_view()),
      path('users/<int:pk>/', views.UserDetail.as_view()),
  ```

## 通过用户操作Snippets

* 现在的Snippet生成中传入了`User`，但是并不是序列化的一部分

* 在`SnippetList`视图中添加`perform_create`操作进行`User`的序列化

  ```python
      def perform_create(self, serializer):
          serializer.save(owner=self.request.user)
  ```

  * 此时生成`Snippet`对象时会自动使用当前的`User`

* 以及获取了`User`信息，还需要在序列化器中定义相关属性

  ```python
      owner = serializers.ReadOnlyField(source='owner.username')    
  ```

  * 记得在`Meta`中添加`fields`
  * 此处的`source`可以直接使用点句法
  * 也可以指定`CharField(read_only=True)`
  * 只读属性在序列化生成时被使用，在删除或修改时不被作用

## 添加视图的权限

* REST提供了一些默认的权限，比如`IsAuthenticatedOrReadOnly`

* 在对应的视图（`SnippetList`和`SnippetDetail`）中添加对应的权限

  ```python
  from rest_framework import permissions
  
  	# in view classes
  	permissions_classes = [permissions.IsAuthenticateOrReadOnly]
  ```

## 在登录界面中添加浏览器URL

* 在项目登记的`urls.py`中添加认证URL

* REST提供了相关的认证界面

* 此处是`rest_tutorial/urls.py`

  ```python
  from django.urls import path, include
  
  urlpatterns += [
  	path('api-auth/', include('rest_framework.urls')),
  ]
  ```

* 当然，`'api-auth'`是一个自定义的路径名称

## 对象级的权限设置

* 在`snippets`应用中，创建一个`permissions.py`文件

  ```python
  from rest_framework import permissions
  
  
  
  class IsOwnerOrReadOnly(permissions.BasePermission):
      '''
      Custom permission to only allow owners of an object to edit it
      '''
  
      def has_object_permission(self, request, view, obj):
          # Read permissions are allowed to any request,
          # so we'll always allow GET, HEAD or OPTIONS requests.
          if request.method in permissions.SAFE_METHODS:
              return True
  
          
          # Write permissions are only allowed to the owner of the snippet.
          return obj.owner == request.user
  ```

* 此时的自定义权限类可以在view中的`permission_classes`中指定

  ```python
  from snippets.permissions import IsOwnerOrReadOnly
  
  	# in views
  	permission_classes = [permissions.IsAuthenticateOrReadOnly, IsOwnerOrReadOnly]
  ```

## 认证测试

* 直接使用`POST`会出现错误

  ```bash
  http POST http://127.0.0.1:8000/snippets/ code="print(123)"
  
  {
      "detail": "Authentication credentials were not provided."
  }
  ```

* 通过`http -a`参数登录再使用相关操作


# 关系和超链接

* 目前位置，API使用主键来作为代表，为了提高API的集成度和可见度，使用超链接进行关联。

## 在API根上创建endpoint

* 在`snippets/views.py`中添加`api_root`

  ```python
  from rest_framework.decorators import api_view
  from rest_framework.response import Response
  from rest_framework.reverse import reverse
  
  
  @api_view(['GET'])
  def api_root(request, format=None):
      return Response({
          'users': reverse('user-list', request=request, format=format),
          'snippets': reverse('snippet-list', request=request, format=format)
      })
  ```

* 使用REST的`reverse`而不是django的

* 此处的urlpattern会在`urls.py`中声明

## 为高亮代码片段创建endpoint

* 此处使用预渲染的HTML

* 在`snippets/views.py`中创建相关的视图，并添加`.get()`方法

  ```python
  from rest_framework import renderers
  
  
  class SnippetHighlight(generics.GenericAPIView):
      queryset = Snippet.objects.all()
      renderer_classes = [renderers.StaticHTMLRenderer]
  
      
      def get(self, request, *args, **kwargs):
          snippet = self.get_object()
          return Response(snippet.highlighted)
  ```

* 在`snippets/urls.py`中添加相关的url

  ```python
  path('', views.api_root),
  path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view())
  ```

## 超链接API

* 表示（Representation）
  * 数据的表现形式或者是**不同的序列化形式**
  * 当用户请求API时，返回的JSON对象就是一个用户**实体（entity）**的**表示**
* 实体（Entity）
  * 数据的实际形式，如类的实例或者数据库的记录
  * 通常包含了一些字段或者属性

## 常见的几种表示关系的方式

1. 使用主键
2. 使用超链接
3. 使用独特的标记属性
4. 使用默认的字符串表示（representation）
5. 在父表示内内联一个相关的实体（entity）
6. 其他的客制化方式进行表示

* django对上述所有的表示方式提供了支持

* 使用超链接进行关联表示时，需要让序列化器继承`HyperlinkedModelSerializer`（而不是现在的`ModelSerializer`）

  * 默认情况下，这个类不具有`id`字段
  * 包含`url`字段，这个字段的类型是`HyperlinkedIdentityField`
  * 关联使用的是`HyperlinkedRelatedField`（而不是现在的`PrimaryKeyRelatedField`）

* 在`snippets/serializers.py`中改写序列化器

  ```python
  class SnippetSerializer(serializers.HyperlinkedModelSerializer):
      owner = serializers.ReadOnlyField(source='owner.username')
      highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')
  
  
      class Meta:
          model = Snippet
          field = ['url', 'id', 'highlight', 'owner', 'title', 'code', 'linenos', 'language', 'style']
  
  
  
  from django.contrib.auth.models import User
  class UserSerializer(serializers.HyperlinkedModelSerializer):
      snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)
  
  
      class Meta:
          model = User
          fields = ['url', 'id', 'username', 'snippets']
  ```

* 这里的`highlight`字段和`url`是同一个类型，但它指向`snippet-highlight`URL而不是`snippet-detail`

* 默认的序列化器自身的`url`字段会指向`MODELNAME-detail`路径

## 添加分页

* 在`tutorial/settings.py`添加分页设置

  ```python
  REST_FRAMEWORK = {
      'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
      'PAGE_SIZE': 10,
  }
  ```

* 所有的REST设置在`REST_FRAMEWOK`中，以便于和其他项目设置分离

# ViewSet和Router

* REST框架提供了ViewSet来自动实现相关的`retrieve`、`update`等方法，而不需要自己实现`get`或`put`方法
* 通常使用`Router`来对复杂的URL进行配置

## 使用ViewSet重写

```python
# Use ViewSet and Router
from rest_framework import viewsets
from django.contrib.auth.models import User
from snippets.serializers import UserSerializer, SnippetSerializer


class UserViewSet(viewsets.ReadOnlyModelViewSet):
    '''
    This viewset automatically provide 'list' and 'retrive' actions.
    '''
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

* 使用`ReadOnlyModelViewSet`来进行默认的Read Only操作

  ```python
  
  # Use ViewSet and Router
  from snippets.models import Snippet
  from rest_framework import viewsets
  from django.contrib.auth.models import User
  from snippets.serializers import UserSerializer, SnippetSerializer
  from rest_framework import permissions
  from rest_framework import renderers
  from rest_framework.decorators import action
  from rest_framework.response import Response
  from snippets.permissions import IsOwnerOrReadOnly
  
  
  
  class UserViewSet(viewsets.ReadOnlyModelViewSet):
      '''
      This viewset automatically provide 'list' and 'retrive' actions.
      '''
      queryset = User.objects.all()
      serializer_class = UserSerializer
      
  
  
  class SnippetViewSet(viewsets.ModelViewSet):
      '''
      This ViewSet automatically provides 'list', 'create', 'retrieve',
      'update' and 'destroy' actions
      
      Additionally we also provide an extra 'highlight' action.
      '''
      queryset = Snippet.objects.all()
      serializer_class = SnippetSerializer
      permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
  
      @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
      def highlight(self, request, *args, **kwargs):
          snippet = self.get_object()
          return Response(snippet.highlight)
      
      def perform_create(self, serializer):
          serializer.save(owner=self.request.user)
  ```

* 使用ViewSet重写Snippet的相关视图

* 通过使用`ModelViewSet`来获取完全的默认读写操作权限

* `@action`修饰符可以用在所有非默认的`create`、`update`、`delete`等方法

  * `@action`修饰符默认接收`GET`请求，可以使用`methods`参数来指定别的请求类型
  * 默认的URL会根据方法名生成，可以使用`url_path`参数来自定义URL

## 显式连接ViewSet和URL

* 在`urls.py`中需要指定各个ViewSet对应的多种请求类型和对应的方法

  ```python
  from rest_framework import renderers
  from snippets.views import api_root, SnippetViewSet, UserViewSet
  
  
  snippet_list = SnippetViewSet.as_view(
      {
          'get': 'list',
          'post': 'create'
      }
  )
  
  
  snippet_detail = SnippetViewSet.as_view(
      {
          'get': 'retrieve',
          'put': 'update',
          'patch': 'partial_update',
          'delete': 'destroy'
      }
  )
  
  
  snippet_highlight = SnippetViewSet.as_view(
      {
          'get': 'highlight'
      }, renderer_classes = [renderers.StaticHTMLRenderer]
  )
  
  
  user_list = UserViewSet.as_view(
      {
          'get': 'list'
      }
  )
  
  
  user_detail = UserViewSet.as_view(
      {
          'get': 'retrieve'
      }
  )
  ```

* 然后才可以进行`urlpatterns`的配置

  ```python
  from rest_framework.urlpatterns import format_suffix_patterns
  urlpatterns = format_suffix_patterns([
      path('', api_root),
      path('snippets/', snippet_list, name='snippet-list'),
      path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
      path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
      path('users/', user_list, name='user-list'),
      path('users/<int:pk>/', user_detail, name='user-detail'),
  ])
  ```

## 使用路由

* 事实上在使用ViewSet时，不需要自己设计URL，REST提供了`Router`来进行URL分配

* 使用路由更新`urls.py`

  ```python
  from django.urls import path, include
  from rest_framework.routers import DefaultRouter
  
  
  from snippets import views
  
  # Create a router and register our ViewSet with it
  router = DefaultRouter()
  router.register(r'snippets', views.SnippetViewSet, basename='snippet')
  router.register (r'users', views.UserViewSet, basename='user')
  
  # The API URLs are now determined automatically by the router.
  urlpatterns = [
      path('', include(router.urls))
  ]
  ```

  
