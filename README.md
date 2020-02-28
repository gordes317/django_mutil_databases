# django_mutil_databases
django 使用多数据源


参考文档
1、https://www.cnblogs.com/wumingxiaoyao/p/8610791.html
2、https://www.cnblogs.com/DJRemix/p/11584563.html

1. 修改项目的 settings 配置 
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    },
    'ora1': {   # 配置第二个数据库节点名称
        'ENGINE': 'django.db.backends.oracle',
        'NAME': 'devdb',
        'USER': 'hysh',
        'PASSWORD': 'hysh',
        'HOST': '192.168.191.3',
        'PORT': '1521',
    },
}

2. 设置数据库的路由规则方法 

在 settings.py 中配置 DATABASE_ROUTERS
DATABASE_ROUTERS = ['Prject.database_router.DatabaseAppsRouter']

Project: 建立的django项目名称(project_name) 
database_router: 定义路由规则database_router.py 文件名称, 这个文件名可以自己定义 
DatabaseAppsRouter: 路由规则的类名称，这个类是在database_router.py 文件中定义

3. 设置APP对应的数据库路由表 

每个APP要连接哪个数据库，需要在做匹配设置，在 settings.py 文件中做如下配置：
DATABASE_APPS_MAPPING = {
    # example:
    # 'app_name':'database_name',
    'app01': 'ora1',
    'admin': 'defualt',
    'app02':  'defualt',
}
以上的app01， app02是项目中的 APP名，分别指定到 ora1， default的数据库。

4. 创建数据库路由规则 
在项目工程根路径下(与 settings.py 文件一级）创建 database_router.py 文件:

from django.conf import settings
 
DATABASE_MAPPING = settings.DATABASE_APPS_MAPPING
 
 
class DatabaseAppsRouter(object):
    """
    A router to control all database operations on models for different
    databases.
 
    In case an app is not set in settings.DATABASE_APPS_MAPPING, the router
    will fallback to the `default` database.
 
    Settings example:
 
    DATABASE_APPS_MAPPING = {'app1': 'db1', 'app2': 'db2'}
    """
 
    def db_for_read(self, model, **hints):
        """"Point all read operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None
 
    def db_for_write(self, model, **hints):
        """Point all write operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None
 
    def allow_relation(self, obj1, obj2, **hints):
        """Allow any relation between apps that use the same database."""
        db_obj1 = DATABASE_MAPPING.get(obj1._meta.app_label)
        db_obj2 = DATABASE_MAPPING.get(obj2._meta.app_label)
        if db_obj1 and db_obj2:
            if db_obj1 == db_obj2:
                return True
            else:
                return False
        return None
 
    def allow_syncdb(self, db, model):
        """Make sure that apps only appear in the related database."""
 
        if db in DATABASE_MAPPING.values():
            return DATABASE_MAPPING.get(model._meta.app_label) == db
        elif model._meta.app_label in DATABASE_MAPPING:
            return False
        return None
 
    def allow_migrate(self, db, app_label, model=None, **hints):
        """
        Make sure the auth app only appears in the 'auth_db'
        database.
        """
        if db in DATABASE_MAPPING.values():
            return DATABASE_MAPPING.get(app_label) == db
        elif app_label in DATABASE_MAPPING:
            return False
        return None
5.原生sql 的使用：
def exc_sql(sql):
    cursor = connections['ora1'].cursor()
    # cursor = connection.cursor()
    cursor.execute(sql)
    result = cursor.fetchall()
    return result
    
6. Models创建样例 
在各自的 APP 中创建数据表的models时，必须要指定表的 app_label 名字，如果不指定则会创建到 default 中配置的数据库名下， 
如下：
在app01下创建models
app01/models.py
class Users(models.Model):
    name = models.CharField(max_length=50)
    passwd = models.CharField(max_length=100)
 
    def __str__(self):
        return "app01 %s " % self.name
 
    class Meta:
        app_label = "app01"
在app02下创建models
app02/models.py
class Users(models.Model):
    username = models.CharField(max_length=100)
    password = models.CharField(max_length=50)
    age = models.IntegerField()
 
    def __str__(self):
        return "app02 %s" % self.username
 
    class Meta:
        app_label = "app02"
 
class Book(models.Model):
    user = models.ForeignKey("Users", on_delete=models.CASCADE)
    bookname = models.CharField(max_length=100)
 
    def __str__(self):
        return "%s: %s" % (self.user.username, self.bookname)
 
    class Meta:
        app_label = "app01"
        
7. 生成数据表 
在使用django的 migrate 创建生成表的时候，需要加上 –database 参数，如果不加则将 未 指定 app_label 的 APP的models中的表创建到default指定的数据库中,如:

将app01下models中的表创建到db01的数据库”db_01”中

./ manage.py  migrate  --database=db01                                
将app02下models中的表创建到db02的数据库”db_02”中

./ manage.py  migrate  --database=db02
将app03下models中的表创建到default的数据库”sqlite3”中

./ manage.py  migrate

8、在Django 的管理站点中使用多数据库
Django 的管理站点没有对多数据库的任何显式的支持。如果你给数据库上某个模型提供的管理站点不想通过你的路由链指定，你将需要编写自定义的ModelAdmin类用来将管理站点导向一个特殊的数据库。
ModelAdmin 对象具有5个方法，它们需要定制以支持多数据库：
在app02/admin.py

from app02.models import Book
from django.contrib import admin

class MultiDBModelAdmin(admin.ModelAdmin):
    # A handy constant for the name of the alternate database.
    using = 'app01'

    def save_model(self, request, obj, form, change):
        # Tell Django to save objects to the 'other' database.
        obj.save(using=self.using)

    def delete_model(self, request, obj):
        # Tell Django to delete objects from the 'other' database
        obj.delete(using=self.using)

    def get_queryset(self, request):
        # Tell Django to look for objects on the 'other' database.
        return super(MultiDBModelAdmin, self).get_queryset(request).using(self.using)

    def formfield_for_foreignkey(self, db_field, request=None, **kwargs):
        # Tell Django to populate ForeignKey widgets using a query
        # on the 'other' database.
        return super(MultiDBModelAdmin, self).formfield_for_foreignkey(db_field, request=request, using=self.using, **kwargs)

    def formfield_for_manytomany(self, db_field, request=None, **kwargs):
        # Tell Django to populate ManyToMany widgets using a query
        # on the 'other' database.
        return super(MultiDBModelAdmin, self).formfield_for_manytomany(db_field, request=request, using=self.using, **kwargs)
        
class BookAdmin(MultiDBModelAdmin):
    list_display = ['user', 'bookname']
admin.site.register(Book, MultiDBModelAdmin)
