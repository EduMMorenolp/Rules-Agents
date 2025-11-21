# Python Django - Patterns & Best Practices

## 🐍 **REGLA OBLIGATORIA: Django Conventions**

### ✅ **Estructura de Proyecto Django**
```python
# ✅ CORRECTO - Estructura estándar
myproject/
├── manage.py
├── myproject/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   ├── products/
│   └── orders/
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
└── tests/
```

## 🏗️ **Model Patterns**

### **Model Definition**
```python
# models.py
from django.db import models
from django.contrib.auth.models import AbstractUser
from django.utils.translation import gettext_lazy as _

class TimeStampedModel(models.Model):
    """Modelo base con timestamps automáticos"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class User(AbstractUser):
    """Usuario personalizado"""
    email = models.EmailField(_('email address'), unique=True)
    first_name = models.CharField(_('first name'), max_length=150)
    is_active = models.BooleanField(_('active'), default=True)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['first_name']
    
    class Meta:
        db_table = 'users'
        verbose_name = _('User')
        verbose_name_plural = _('Users')

class Product(TimeStampedModel):
    """Modelo de producto"""
    name = models.CharField(_('name'), max_length=200)
    slug = models.SlugField(_('slug'), unique=True)
    price = models.DecimalField(_('price'), max_digits=10, decimal_places=2)
    is_active = models.BooleanField(_('active'), default=True)
    
    class Meta:
        db_table = 'products'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['is_active', 'created_at']),
        ]
```

### **Manager Patterns**
```python
# managers.py
from django.db import models

class ActiveManager(models.Manager):
    """Manager para objetos activos"""
    def get_queryset(self):
        return super().get_queryset().filter(is_active=True)

class ProductManager(models.Manager):
    """Manager personalizado para productos"""
    def active(self):
        return self.filter(is_active=True)
    
    def by_category(self, category):
        return self.active().filter(category=category)
    
    def featured(self):
        return self.active().filter(is_featured=True)

# En models.py
class Product(TimeStampedModel):
    # ... campos
    
    objects = models.Manager()  # Manager por defecto
    active = ActiveManager()    # Manager para activos
    products = ProductManager() # Manager personalizado
```

## 🎯 **View Patterns**

### **Class-Based Views**
```python
# views.py
from django.views.generic import ListView, DetailView, CreateView
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import JsonResponse
from django.shortcuts import get_object_or_404

class ProductListView(ListView):
    """Lista de productos con paginación"""
    model = Product
    template_name = 'products/list.html'
    context_object_name = 'products'
    paginate_by = 20
    
    def get_queryset(self):
        return Product.active.all().select_related('category')
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        return context

class ProductDetailView(DetailView):
    """Detalle de producto"""
    model = Product
    template_name = 'products/detail.html'
    context_object_name = 'product'
    
    def get_object(self):
        return get_object_or_404(
            Product.active.select_related('category'),
            slug=self.kwargs['slug']
        )

class ProductCreateView(LoginRequiredMixin, CreateView):
    """Crear producto (solo usuarios autenticados)"""
    model = Product
    fields = ['name', 'description', 'price', 'category']
    template_name = 'products/create.html'
    
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)
```

### **API Views (DRF)**
```python
# api/views.py
from rest_framework import generics, status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from django.db import transaction

class ProductListCreateAPIView(generics.ListCreateAPIView):
    """API para listar y crear productos"""
    queryset = Product.active.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        queryset = super().get_queryset()
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category__slug=category)
        return queryset
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

@api_view(['POST'])
@permission_classes([IsAuthenticated])
def bulk_update_products(request):
    """Actualización masiva de productos"""
    try:
        with transaction.atomic():
            for item in request.data.get('products', []):
                product = Product.objects.get(id=item['id'])
                product.price = item['price']
                product.save()
        
        return Response({'status': 'success'})
    except Exception as e:
        return Response(
            {'error': str(e)}, 
            status=status.HTTP_400_BAD_REQUEST
        )
```

## 📋 **Serializer Patterns**

### **DRF Serializers**
```python
# serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model

User = get_user_model()

class UserSerializer(serializers.ModelSerializer):
    """Serializer para usuario"""
    full_name = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'email', 'first_name', 'last_name', 'full_name']
        read_only_fields = ['id']
    
    def get_full_name(self, obj):
        return f"{obj.first_name} {obj.last_name}".strip()

class ProductSerializer(serializers.ModelSerializer):
    """Serializer para producto"""
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'slug', 'price', 'category', 'category_name']
        read_only_fields = ['id', 'slug']
    
    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError("El precio debe ser mayor a 0")
        return value
    
    def create(self, validated_data):
        # Generar slug automáticamente
        validated_data['slug'] = slugify(validated_data['name'])
        return super().create(validated_data)
```

## 🔧 **Settings Configuration**

### **Settings Structure**
```python
# settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

# Configuración básica
SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = False
ALLOWED_HOSTS = []

# Aplicaciones
DJANGO_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

THIRD_PARTY_APPS = [
    'rest_framework',
    'corsheaders',
    'django_extensions',
]

LOCAL_APPS = [
    'apps.users',
    'apps.products',
    'apps.orders',
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# Middleware
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Base de datos
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}

# Internacionalización
LANGUAGE_CODE = 'es-es'
TIME_ZONE = 'America/Argentina/Buenos_Aires'
USE_I18N = True
USE_TZ = True

# Usuario personalizado
AUTH_USER_MODEL = 'users.User'
```

### **Development Settings**
```python
# settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

# Base de datos para desarrollo
DATABASES['default']['NAME'] = 'myproject_dev'

# Email backend para desarrollo
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Django Debug Toolbar
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']

INTERNAL_IPS = ['127.0.0.1']
```

## 🧪 **Testing Patterns**

### **Model Tests**
```python
# tests/test_models.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from apps.products.models import Product, Category

User = get_user_model()

class ProductModelTest(TestCase):
    """Tests para el modelo Product"""
    
    def setUp(self):
        self.user = User.objects.create_user(
            email='test@example.com',
            password='testpass123'
        )
        self.category = Category.objects.create(name='Electronics')
    
    def test_product_creation(self):
        """Test creación de producto"""
        product = Product.objects.create(
            name='Test Product',
            price=99.99,
            category=self.category,
            created_by=self.user
        )
        
        self.assertEqual(product.name, 'Test Product')
        self.assertEqual(product.slug, 'test-product')
        self.assertTrue(product.is_active)
    
    def test_product_str_representation(self):
        """Test representación string del producto"""
        product = Product.objects.create(
            name='Test Product',
            price=99.99,
            category=self.category
        )
        
        self.assertEqual(str(product), 'Test Product')
```

### **View Tests**
```python
# tests/test_views.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model

User = get_user_model()

class ProductViewTest(TestCase):
    """Tests para vistas de productos"""
    
    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user(
            email='test@example.com',
            password='testpass123'
        )
    
    def test_product_list_view(self):
        """Test vista de lista de productos"""
        response = self.client.get(reverse('products:list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Products')
    
    def test_product_create_requires_login(self):
        """Test que crear producto requiere login"""
        response = self.client.get(reverse('products:create'))
        
        self.assertEqual(response.status_code, 302)  # Redirect to login
    
    def test_product_create_authenticated(self):
        """Test crear producto autenticado"""
        self.client.login(email='test@example.com', password='testpass123')
        response = self.client.get(reverse('products:create'))
        
        self.assertEqual(response.status_code, 200)
```

## 🔒 **Security Best Practices**

### **Authentication & Authorization**
```python
# utils/permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """Permiso personalizado para propietarios"""
    
    def has_object_permission(self, request, view, obj):
        # Permisos de lectura para cualquiera
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Permisos de escritura solo para el propietario
        return obj.created_by == request.user

# Uso en views
class ProductViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
```

### **Input Validation**
```python
# validators.py
from django.core.exceptions import ValidationError
from django.utils.translation import gettext_lazy as _

def validate_positive_price(value):
    """Validar que el precio sea positivo"""
    if value <= 0:
        raise ValidationError(
            _('El precio debe ser mayor a 0'),
            code='invalid_price'
        )

def validate_file_size(value):
    """Validar tamaño de archivo"""
    limit = 5 * 1024 * 1024  # 5MB
    if value.size > limit:
        raise ValidationError(
            _('El archivo no puede ser mayor a 5MB'),
            code='file_too_large'
        )
```

Esta configuración garantiza desarrollo Django siguiendo las mejores prácticas y patrones de la comunidad.