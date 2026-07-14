#!/bin/bash

echo "🚛 Создание проекта 'Грузавтотранс' с картой, весом и адресами..."

# Создаём структуру папок
mkdir -p gruzavtotrans/backend/config
mkdir -p gruzavtotrans/backend/apps/{users,documents,orders,chat,notifications}
mkdir -p gruzavtotrans/frontend/public
mkdir -p gruzavtotrans/frontend/src/components/{auth,dashboard,profile,orders,chat,map}
mkdir -p gruzavtotrans/frontend/src/context
mkdir -p gruzavtotrans/frontend/src/hooks
mkdir -p gruzavtotrans/frontend/src/services

cd gruzavtotrans

# ======================== BACKEND ========================

# ---- backend/requirements.txt ----
cat > backend/requirements.txt << 'EOF'
Django==4.2.7
djangorestframework==3.14.0
django-cors-headers==4.3.1
channels==4.0.0
channels-redis==4.1.0
celery==5.3.4
redis==5.0.1
psycopg2-binary==2.9.9
django-environ==0.11.2
django-storages==1.14.2
boto3==1.34.11
twilio==8.8.0
sendgrid==6.10.0
EOF

# ---- backend/Dockerfile ----
cat > backend/Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"]
EOF

# ---- backend/manage.py ----
cat > backend/manage.py << 'EOF'
#!/usr/bin/env python
import os
import sys

def main():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed?"
        ) from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
EOF

# ---- backend/config/settings.py ----
cat > backend/config/settings.py << 'EOF'
import os
from pathlib import Path
import environ

env = environ.Env()
BASE_DIR = Path(__file__).resolve().parent.parent
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

SECRET_KEY = env('SECRET_KEY', default='django-insecure-123456')
DEBUG = env.bool('DEBUG', default=True)
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=['*'])

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'corsheaders',
    'channels',
    'apps.users',
    'apps.documents',
    'apps.orders',
    'apps.chat',
    'apps.notifications',
]

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

ROOT_URLCONF = 'config.urls'
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
WSGI_APPLICATION = 'config.wsgi.application'
ASGI_APPLICATION = 'config.asgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME', default='gruz_db'),
        'USER': env('DB_USER', default='gruz_user'),
        'PASSWORD': env('DB_PASSWORD', default='password'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
    }
}

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {'hosts': [(env('REDIS_HOST', default='localhost'), 6379)]},
    }
}

CORS_ALLOW_ALL_ORIGINS = True
STATIC_URL = '/static/'
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

TWILIO_ACCOUNT_SID = env('TWILIO_ACCOUNT_SID', default='')
TWILIO_AUTH_TOKEN = env('TWILIO_AUTH_TOKEN', default='')
TWILIO_PHONE = env('TWILIO_PHONE', default='')

EMAIL_HOST = 'smtp.sendgrid.net'
EMAIL_HOST_USER = env('EMAIL_HOST_USER', default='')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD', default='')
EMAIL_PORT = 587
EMAIL_USE_TLS = True

CELERY_BROKER_URL = f"redis://{env('REDIS_HOST', default='localhost')}:6379/0"
CELERY_RESULT_BACKEND = CELERY_BROKER_URL
YANDEX_MAPS_API_KEY = env('YANDEX_MAPS_API_KEY', default='')
EOF

# ---- backend/config/urls.py ----
cat > backend/config/urls.py << 'EOF'
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/auth/', include('apps.users.urls')),
    path('api/orders/', include('apps.orders.urls')),
    path('api/documents/', include('apps.documents.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
EOF

# ---- backend/config/asgi.py ----
cat > backend/config/asgi.py << 'EOF'
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from apps.chat.routing import websocket_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
})
EOF

# ---- backend/config/celery.py ----
cat > backend/config/celery.py << 'EOF'
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
EOF

touch backend/config/__init__.py

# ===== apps/users =====
cat > backend/apps/users/models.py << 'EOF'
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    phone = models.CharField(max_length=20, unique=True)
    email = models.EmailField(unique=True)
    full_name = models.CharField(max_length=255)
    inn = models.CharField(max_length=12)
    passport_data = models.JSONField(default=dict)
    registration_address = models.TextField()
    driver_license_data = models.JSONField(default=dict, blank=True)
    role = models.CharField(max_length=20, choices=[('carrier','Перевозчик'),('customer','Заказчик'),('admin','Админ')])
    is_phone_verified = models.BooleanField(default=False)
    is_email_verified = models.BooleanField(default=False)
    is_active = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    fcm_token = models.CharField(max_length=255, blank=True, null=True)

class VerificationCode(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    code = models.CharField(max_length=6)
    type = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    is_used = models.BooleanField(default=False)
EOF

cat > backend/apps/users/serializers.py << 'EOF'
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)
    class Meta:
        model = User
        fields = ('id', 'email', 'phone', 'full_name', 'inn', 'passport_data',
                  'registration_address', 'driver_license_data', 'role', 'password')
    def create(self, validated_data):
        user = User(**validated_data)
        user.set_password(validated_data['password'])
        user.save()
        return user
EOF

cat > backend/apps/users/views.py << 'EOF'
import random
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from .models import User, VerificationCode
from .serializers import UserSerializer
from apps.notifications.tasks import send_sms, send_email

class RegisterView(APIView):
    permission_classes = [AllowAny]
    def post(self, request):
        serializer = UserSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save(is_active=False)
            sms_code = f"{random.randint(100000, 999999)}"
            VerificationCode.objects.create(user=user, code=sms_code, type='sms')
            send_sms.delay(user.phone, f'Код подтверждения: {sms_code}')
            return Response({'message': 'SMS отправлен', 'phone': user.phone}, status=201)
        return Response(serializer.errors, status=400)

class VerifySMSView(APIView):
    permission_classes = [AllowAny]
    def post(self, request):
        phone = request.data.get('phone')
        code = request.data.get('code')
        try:
            user = User.objects.get(phone=phone)
            vc = VerificationCode.objects.filter(user=user, code=code, type='sms', is_used=False).first()
            if vc:
                vc.is_used = True
                vc.save()
                user.is_phone_verified = True
                user.save()
                email_code = f"{random.randint(100000, 999999)}"
                VerificationCode.objects.create(user=user, code=email_code, type='email')
                send_email.delay(user.email, 'Подтверждение email', f'Ваш код: {email_code}')
                return Response({'message': 'Телефон подтверждён, проверьте почту', 'email': user.email})
        except User.DoesNotExist:
            pass
        return Response({'error': 'Неверный код или телефон'}, status=400)

class VerifyEmailView(APIView):
    permission_classes = [AllowAny]
    def post(self, request):
        email = request.data.get('email')
        code = request.data.get('code')
        try:
            user = User.objects.get(email=email)
            vc = VerificationCode.objects.filter(user=user, code=code, type='email', is_used=False).first()
            if vc:
                vc.is_used = True
                vc.save()
                user.is_email_verified = True
                user.is_active = True
                user.save()
                return Response({'message': 'Регистрация завершена'})
        except User.DoesNotExist:
            pass
        return Response({'error': 'Неверный код или email'}, status=400)
EOF

cat > backend/apps/users/urls.py << 'EOF'
from django.urls import path
from .views import RegisterView, VerifySMSView, VerifyEmailView

urlpatterns = [
    path('register/', RegisterView.as_view()),
    path('verify-sms/', VerifySMSView.as_view()),
    path('verify-email/', VerifyEmailView.as_view()),
]
EOF

# ===== apps/documents =====
cat > backend/apps/documents/models.py << 'EOF'
from django.db import models
from apps.users.models import User

class Document(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    type = models.CharField(max_length=50)
    file = models.FileField(upload_to='documents/%Y/%m/%d/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
    verified = models.BooleanField(default=False)
EOF

# ===== apps/orders =====
cat > backend/apps/orders/models.py << 'EOF'
from django.db import models
from apps.users.models import User

class Address(models.Model):
    street = models.CharField(max_length=255)
    house = models.CharField(max_length=20)
    apartment = models.CharField(max_length=20, blank=True)
    city = models.CharField(max_length=100)
    region = models.CharField(max_length=100, blank=True)
    country = models.CharField(max_length=100, default='Россия')
    latitude = models.FloatField(null=True, blank=True)
    longitude = models.FloatField(null=True, blank=True)

    def __str__(self):
        return f"{self.city}, {self.street} {self.house}"

class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders_as_customer')
    carrier = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, blank=True, related_name='orders_as_carrier')
    pickup_address = models.ForeignKey(Address, on_delete=models.CASCADE, related_name='pickup_orders')
    delivery_address = models.ForeignKey(Address, on_delete=models.CASCADE, related_name='delivery_orders')
    cargo_description = models.TextField()
    weight = models.FloatField()
    volume = models.FloatField(default=0)
    date_required = models.DateField()
    status = models.CharField(max_length=20, default='created',
                              choices=[('created','Создан'),('searching','Ищет перевозчика'),
                                       ('accepted','Принят'),('in_progress','В пути'),
                                       ('delivered','Доставлен'),('cancelled','Отменён')])
    public_url = models.CharField(max_length=100, unique=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"Заказ #{self.id} ({self.weight}т)"
EOF

cat > backend/apps/orders/views.py << 'EOF'
from rest_framework.generics import ListCreateAPIView, RetrieveAPIView
from rest_framework.permissions import IsAuthenticated
from .models import Order, Address
from .serializers import OrderSerializer, AddressSerializer

class OrderListCreateView(ListCreateAPIView):
    serializer_class = OrderSerializer
    permission_classes = [IsAuthenticated]
    def get_queryset(self):
        return Order.objects.all()

class OrderDetailView(RetrieveAPIView):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
EOF

cat > backend/apps/orders/serializers.py << 'EOF'
from rest_framework import serializers
from .models import Order, Address

class AddressSerializer(serializers.ModelSerializer):
    class Meta:
        model = Address
        fields = '__all__'

class OrderSerializer(serializers.ModelSerializer):
    pickup_address = AddressSerializer()
    delivery_address = AddressSerializer()

    class Meta:
        model = Order
        fields = '__all__'

    def create(self, validated_data):
        pickup_data = validated_data.pop('pickup_address')
        delivery_data = validated_data.pop('delivery_address')
        pickup = Address.objects.create(**pickup_data)
        delivery = Address.objects.create(**delivery_data)
        return Order.objects.create(pickup_address=pickup, delivery_address=delivery, **validated_data)
EOF

cat > backend/apps/orders/urls.py << 'EOF'
from django.urls import path
from .views import OrderListCreateView, OrderDetailView

urlpatterns = [
    path('', OrderListCreateView.as_view()),
    path('<int:pk>/', OrderDetailView.as_view()),
]
EOF

# ===== apps/chat =====
cat > backend/apps/chat/consumers.py << 'EOF'
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.order_id = self.scope['url_route']['kwargs']['order_id']
        self.room_group_name = f'chat_{self.order_id}'
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        await self.channel_layer.group_send(
            self.room_group_name,
            {'type': 'chat_message', 'message': data['message']}
        )

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({'message': event['message']}))
EOF

cat > backend/apps/chat/routing.py << 'EOF'
from django.urls import re_path
from .consumers import ChatConsumer

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<order_id>\w+)/$', ChatConsumer.as_asgi()),
]
EOF

# ===== apps/notifications =====
cat > backend/apps/notifications/tasks.py << 'EOF'
from celery import shared_task

@shared_task
def send_sms(phone, message):
    print(f'SMS to {phone}: {message}')
    return True

@shared_task
def send_email(recipient, subject, body):
    print(f'Email to {recipient}: {subject} - {body}')
    return True
EOF

touch backend/apps/notifications/services.py

# Создаём __init__.py для всех приложений
for app in users documents orders chat notifications; do
    touch backend/apps/$app/__init__.py
done

# ======================== FRONTEND ========================

# ---- frontend/package.json ----
cat > frontend/package.json << 'EOF'
{
  "name": "gruzavtotrans-frontend",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "axios": "^1.6.2",
    "react-yandex-maps": "^4.0.0",
    "tailwindcss": "^3.3.6"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.0.0"
  }
}
EOF

cat > frontend/vite.config.js << 'EOF'
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8000',
      '/ws': { target: 'ws://localhost:8000', ws: true },
    },
  },
})
EOF

cat > frontend/tailwind.config.js << 'EOF'
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: { extend: {} },
  plugins: [],
}
EOF

cat > frontend/public/index.html << 'EOF'
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Грузавтотранс</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/index.js"></script>
</body>
</html>
EOF

cat > frontend/src/index.css << 'EOF'
@tailwind base;
@tailwind components;
@tailwind utilities;
EOF

cat > frontend/src/index.js << 'EOF'
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode><App /></React.StrictMode>
);
EOF

# ---- frontend/src/App.jsx ----
cat > frontend/src/App.jsx << 'EOF'
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import Register from './components/auth/Register';
import Login from './components/auth/Login';
import Dashboard from './components/dashboard/Dashboard';
import OrderForm from './components/orders/OrderForm';
import OrderMap from './components/orders/OrderMap';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/register" element={<Register />} />
          <Route path="/login" element={<Login />} />
          <Route path="/" element={<Dashboard />} />
          <Route path="/order/new" element={<OrderForm />} />
          <Route path="/order/:slug" element={<OrderMap />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}
export default App;
EOF

# ---- frontend/src/components/auth/Register.jsx (пошаговый) ----
cat > frontend/src/components/auth/Register.jsx << 'EOF'
import { useState } from 'react';
import api from '../../services/api';

export default function Register() {
  const [step, setStep] = useState('form');
  const [form, setForm] = useState({});
  const [phone, setPhone] = useState('');
  const [email, setEmail] = useState('');
  const [smsCode, setSmsCode] = useState('');
  const [emailCode, setEmailCode] = useState('');

  const handleChange = (e) => setForm({ ...form, [e.target.name]: e.target.value });

  const handleRegister = async (e) => {
    e.preventDefault();
    try {
      const res = await api.post('/auth/register/', form);
      setPhone(res.data.phone);
      setStep('sms');
    } catch (err) { alert('Ошибка регистрации'); }
  };

  const handleVerifySms = async () => {
    try {
      const res = await api.post('/auth/verify-sms/', { phone, code: smsCode });
      setEmail(res.data.email);
      setStep('email');
    } catch { alert('Неверный SMS-код'); }
  };

  const handleVerifyEmail = async () => {
    try {
      await api.post('/auth/verify-email/', { email, code: emailCode });
      setStep('done');
    } catch { alert('Неверный email-код'); }
  };

  return (
    <div className="max-w-lg mx-auto bg-white p-8 rounded-2xl shadow mt-10">
      {step === 'form' && (
        <form onSubmit={handleRegister}>
          <h2 className="text-2xl font-bold mb-6">Регистрация перевозчика</h2>
          <input name="full_name" onChange={handleChange} placeholder="ФИО" className="border p-2 w-full mb-2" required />
          <input name="phone" onChange={handleChange} placeholder="Телефон" className="border p-2 w-full mb-2" required />
          <input name="email" type="email" onChange={handleChange} placeholder="Email" className="border p-2 w-full mb-2" required />
          <input name="inn" onChange={handleChange} placeholder="ИНН" className="border p-2 w-full mb-2" />
          <input name="passport_data" onChange={handleChange} placeholder="Паспорт (серия номер)" className="border p-2 w-full mb-2" />
          <input name="registration_address" onChange={handleChange} placeholder="Прописка" className="border p-2 w-full mb-2" />
          <input name="driver_license_data" onChange={handleChange} placeholder="В/У (серия номер)" className="border p-2 w-full mb-2" />
          <input name="vehicle" onChange={handleChange} placeholder="Тягач (госномер)" className="border p-2 w-full mb-2" />
          <input name="trailer" onChange={handleChange} placeholder="Прицеп (госномер)" className="border p-2 w-full mb-2" />
          <button type="submit" className="bg-blue-600 text-white p-2 w-full rounded-xl">Зарегистрироваться</button>
        </form>
      )}
      {step === 'sms' && (
        <div>
          <h2 className="text-xl font-bold">Подтверждение телефона</h2>
          <p>Код отправлен на {phone}</p>
          <input value={smsCode} onChange={(e) => setSmsCode(e.target.value)} placeholder="Введите код" className="border p-2 w-full my-2" />
          <button onClick={handleVerifySms} className="bg-green-600 text-white p-2 w-full rounded-xl">Подтвердить</button>
        </div>
      )}
      {step === 'email' && (
        <div>
          <h2 className="text-xl font-bold">Подтверждение почты</h2>
          <p>Код отправлен на {email}</p>
          <input value={emailCode} onChange={(e) => setEmailCode(e.target.value)} placeholder="Введите код" className="border p-2 w-full my-2" />
          <button onClick={handleVerifyEmail} className="bg-green-600 text-white p-2 w-full rounded-xl">Подтвердить</button>
        </div>
      )}
      {step === 'done' && (
        <div>
          <h2 className="text-2xl font-bold text-green-600">Регистрация завершена!</h2>
          <p>Теперь вы можете войти.</p>
        </div>
      )}
    </div>
  );
}
EOF

# ---- frontend/src/components/auth/Login.jsx (заглушка) ----
cat > frontend/src/components/auth/Login.jsx << 'EOF'
export default function Login() { return <div className="p-6">Страница входа</div>; }
EOF

# ---- frontend/src/components/dashboard/Dashboard.jsx (с картой и статистикой) ----
cat > frontend/src/components/dashboard/Dashboard.jsx << 'EOF'
import { useState, useEffect } from 'react';
import NavigationMap from '../map/NavigationMap';
import api from '../../services/api';

export default function Dashboard() {
  const [stats, setStats] = useState({ carriers: 248, customers: 91, activeOrders: 37, revenue: 4800000 });
  const [orders, setOrders] = useState([]);

  useEffect(() => {
    // В реальном приложении здесь будет запрос к API
    // Пока используем тестовые данные с координатами
    setOrders([
      {
        id: 1,
        weight: 20,
        status: 'created',
        pickup_address: {
          city: 'Москва',
          street: 'Ленина',
          house: '10',
          latitude: 55.7558,
          longitude: 37.6173
        }
      },
      {
        id: 2,
        weight: 12,
        status: 'in_progress',
        pickup_address: {
          city: 'Казань',
          street: 'Баумана',
          house: '5',
          latitude: 55.7963,
          longitude: 49.1088
        }
      },
      {
        id: 3,
        weight: 35,
        status: 'searching',
        pickup_address: {
          city: 'Санкт-Петербург',
          street: 'Невский проспект',
          house: '22',
          latitude: 59.9343,
          longitude: 30.3351
        }
      }
    ]);
  }, []);

  return (
    <div className="min-h-screen bg-gray-100 p-6">
      <h1 className="text-3xl font-bold mb-6">Панель директора</h1>
      <div className="grid grid-cols-4 gap-6 mb-8">
        <div className="bg-white p-4 rounded shadow"><h2>Перевозчики</h2><p className="text-2xl font-bold">{stats.carriers}</p></div>
        <div className="bg-white p-4 rounded shadow"><h2>Заказчики</h2><p className="text-2xl font-bold">{stats.customers}</p></div>
        <div className="bg-white p-4 rounded shadow"><h2>Активные рейсы</h2><p className="text-2xl font-bold">{stats.activeOrders}</p></div>
        <div className="bg-white p-4 rounded shadow"><h2>Доход</h2><p className="text-2xl font-bold">₽ {stats.revenue.toLocaleString()}</p></div>
      </div>
      <div className="bg-white rounded-2xl shadow p-4">
        <h2 className="text-xl font-bold mb-4">Доступные заказы на карте</h2>
        <NavigationMap orders={orders} />
      </div>
    </div>
  );
}
EOF

# ---- frontend/src/components/map/NavigationMap.jsx (главная карта с весом и адресами) ----
cat > frontend/src/components/map/NavigationMap.jsx << 'EOF'
import { useState, useEffect } from 'react';
import { YMaps, Map, Placemark, RoutePanel, ZoomControl } from 'react-yandex-maps';

export default function NavigationMap({ orders = [] }) {
  const [userLocation, setUserLocation] = useState(null);
  const [selectedOrder, setSelectedOrder] = useState(null);

  useEffect(() => {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (pos) => setUserLocation([pos.coords.latitude, pos.coords.longitude]),
        () => console.warn('Геолокация недоступна')
      );
    }
  }, []);

  const mapState = {
    center: userLocation || [55.75, 37.57],
    zoom: 10,
    controls: ['zoomControl', 'fullscreenControl']
  };

  return (
    <YMaps query={{ apikey: import.meta.env.VITE_YANDEX_MAPS_KEY || '' }}>
      <Map state={mapState} width="100%" height="500px" modules={['geoObject.addon.balloon', 'geoObject.addon.hint']}>
        <ZoomControl />
        {orders.map(order => (
          <Placemark
            key={order.id}
            geometry={[order.pickup_address.latitude, order.pickup_address.longitude]}
            properties={{
              balloonContent: `
                <div style="font-size:14px;">
                  <b>Вес:</b> ${order.weight} т<br/>
                  <b>Адрес загрузки:</b> ${order.pickup_address.city}, ул. ${order.pickup_address.street} ${order.pickup_address.house}<br/>
                  <b>Статус:</b> ${order.status}<br/>
                  <button onclick="window.location.href='/order/${order.id}'" style="background:#3b82f6;color:white;padding:4px 12px;border-radius:8px;border:none;margin-top:6px;">Подробнее</button>
                </div>
              `,
              hintContent: `${order.weight}т - ${order.pickup_address.city}, ${order.pickup_address.street} ${order.pickup_address.house}`
            }}
            options={{
              preset: 'islands#blueCircleIcon',
              iconColor: order.status === 'created' ? '#3b82f6' : '#f59e0b'
            }}
            onClick={() => setSelectedOrder(order)}
          />
        ))}
        {selectedOrder && userLocation && (
          <RoutePanel
            options={{ showRouteButton: true, routePanelState: { from: userLocation, to: [selectedOrder.pickup_address.latitude, selectedOrder.pickup_address.longitude] } }}
          />
        )}
      </Map>
    </YMaps>
  );
}
EOF

# ---- frontend/src/components/orders/OrderForm.jsx (с весом и адресами) ----
cat > frontend/src/components/orders/OrderForm.jsx << 'EOF'
import { useState } from 'react';
import api from '../../services/api';

export default function OrderForm() {
  const [form, setForm] = useState({
    pickup_address: { street: '', house: '', city: '' },
    delivery_address: { street: '', house: '', city: '' },
    cargo_description: '',
    weight: '',
    volume: '',
    date_required: ''
  });

  const handleChange = (e) => {
    const { name, value } = e.target;
    if (name.includes('.')) {
      const [parent, child] = name.split('.');
      setForm(prev => ({
        ...prev,
        [parent]: { ...prev[parent], [child]: value }
      }));
    } else {
      setForm(prev => ({ ...prev, [name]: value }));
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await api.post('/orders/', form);
      alert('Заказ создан!');
    } catch (err) {
      alert('Ошибка создания заказа');
    }
  };

  return (
    <div className="max-w-2xl mx-auto bg-white p-6 rounded-2xl shadow mt-10">
      <h2 className="text-2xl font-bold mb-6">Новый заказ</h2>
      <form onSubmit={handleSubmit}>
        <div className="grid grid-cols-2 gap-4">
          <div className="col-span-2">
            <h3 className="font-semibold">Адрес загрузки</h3>
            <input name="pickup_address.city" onChange={handleChange} placeholder="Город" className="border p-2 w-full mb-2" />
            <input name="pickup_address.street" onChange={handleChange} placeholder="Улица" className="border p-2 w-full mb-2" />
            <input name="pickup_address.house" onChange={handleChange} placeholder="Дом" className="border p-2 w-full" />
          </div>
          <div className="col-span-2">
            <h3 className="font-semibold">Адрес доставки</h3>
            <input name="delivery_address.city" onChange={handleChange} placeholder="Город" className="border p-2 w-full mb-2" />
            <input name="delivery_address.street" onChange={handleChange} placeholder="Улица" className="border p-2 w-full mb-2" />
            <input name="delivery_address.house" onChange={handleChange} placeholder="Дом" className="border p-2 w-full" />
          </div>
          <div>
            <label>Вес (тонн)</label>
            <input name="weight" type="number" step="0.1" onChange={handleChange} placeholder="Вес" className="border p-2 w-full" />
          </div>
          <div>
            <label>Объём (м³)</label>
            <input name="volume" type="number" step="0.1" onChange={handleChange} placeholder="Объём" className="border p-2 w-full" />
          </div>
          <div className="col-span-2">
            <label>Описание груза</label>
            <textarea name="cargo_description" onChange={handleChange} placeholder="Описание" className="border p-2 w-full" rows="2" />
          </div>
          <div className="col-span-2">
            <label>Дата доставки</label>
            <input name="date_required" type="date" onChange={handleChange} className="border p-2 w-full" />
          </div>
        </div>
        <button type="submit" className="mt-4 bg-blue-600 text-white p-2 w-full rounded-xl">Создать заказ</button>
      </form>
    </div>
  );
}
EOF

# ---- frontend/src/components/orders/OrderMap.jsx (публичная страница с картой) ----
cat > frontend/src/components/orders/OrderMap.jsx << 'EOF'
import { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import { YMaps, Map, Placemark, RoutePanel } from 'react-yandex-maps';
import api from '../../services/api';

export default function OrderMap() {
  const { slug } = useParams();
  const [order, setOrder] = useState(null);

  useEffect(() => {
    // Для демонстрации используем статичные данные, в реальности будет запрос
    setOrder({
      id: 1,
      weight: 20,
      status: 'created',
      pickup_address: { city: 'Москва', street: 'Ленина', house: '10', latitude: 55.7558, longitude: 37.6173 },
      delivery_address: { city: 'Санкт-Петербург', street: 'Невский', house: '22', latitude: 59.9343, longitude: 30.3351 }
    });
  }, [slug]);

  if (!order) return <div>Загрузка...</div>;

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold">Заказ #{order.id}</h1>
      <p>Вес: {order.weight} т</p>
      <p>Статус: {order.status}</p>
      <p>Загрузка: {order.pickup_address.city}, ул. {order.pickup_address.street} {order.pickup_address.house}</p>
      <p>Доставка: {order.delivery_address.city}, ул. {order.delivery_address.street} {order.delivery_address.house}</p>
      <div className="h-96 mt-4">
        <YMaps query={{ apikey: import.meta.env.VITE_YANDEX_MAPS_KEY || '' }}>
          <Map state={{ center: [order.pickup_address.latitude, order.pickup_address.longitude], zoom: 10 }} width="100%" height="100%">
            <Placemark geometry={[order.pickup_address.latitude, order.pickup_address.longitude]} properties={{ hintContent: 'Загрузка' }} />
            <Placemark geometry={[order.delivery_address.latitude, order.delivery_address.longitude]} properties={{ hintContent: 'Доставка' }} />
            <RoutePanel options={{ showRouteButton: true }}>
              <RoutePanel.ReferencePoint address={`${order.pickup_address.city}, ${order.pickup_address.street} ${order.pickup_address.house}`} />
              <RoutePanel.ReferencePoint address={`${order.delivery_address.city}, ${order.delivery_address.street} ${order.delivery_address.house}`} />
            </RoutePanel>
          </Map>
        </YMaps>
      </div>
    </div>
  );
}
EOF

# ---- Остальные компоненты (заглушки) ----
touch frontend/src/components/auth/VerifySMS.jsx
touch frontend/src/components/auth/VerifyEmail.jsx
touch frontend/src/components/dashboard/StatsCard.jsx
touch frontend/src/components/dashboard/OrdersList.jsx
touch frontend/src/components/dashboard/Notifications.jsx
touch frontend/src/components/profile/Profile.jsx
touch frontend/src/components/profile/DocumentsUpload.jsx
touch frontend/src/components/orders/OrderCard.jsx
touch frontend/src/components/chat/ChatWindow.jsx
touch frontend/src/context/AuthContext.jsx
touch frontend/src/context/OrderContext.jsx
touch frontend/src/hooks/useAuth.js
touch frontend/src/hooks/useOrders.js
touch frontend/src/services/api.js
touch frontend/src/services/websocket.js

# ---- frontend/src/services/api.js (базовый клиент) ----
cat > frontend/src/services/api.js << 'EOF'
import axios from 'axios';
const api = axios.create({ baseURL: '/api' });
export default api;
EOF

# ---- frontend/src/context/AuthContext.jsx ----
cat > frontend/src/context/AuthContext.jsx << 'EOF'
import { createContext, useState } from 'react';
export const AuthContext = createContext();
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return <AuthContext.Provider value={{ user, setUser }}>{children}</AuthContext.Provider>;
}
EOF

# ======================== DOCKER-COMPOSE ========================

cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: gruz_db
      POSTGRES_USER: gruz_user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  redis:
    image: redis:7
    ports:
      - "6379:6379"
  backend:
    build: ./backend
    command: sh -c "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
      - DEBUG=1
  frontend:
    build: ./frontend
    command: npm run dev -- --host
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
    depends_on:
      - backend
volumes:
  postgres_data:
EOF

# ---- frontend/Dockerfile ----
cat > frontend/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev", "--", "--host"]
EOF

# ======================== .env.example и README ========================

cat > .env.example << 'EOF'
SECRET_KEY=your-secret-key
DEBUG=1
DB_NAME=gruz_db
DB_USER=gruz_user
DB_PASSWORD=password
DB_HOST=localhost
REDIS_HOST=localhost
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE=+1234567890
EMAIL_HOST_USER=your-sendgrid-user
EMAIL_HOST_PASSWORD=your-sendgrid-password
YANDEX_MAPS_API_KEY=your-yandex-key
EOF

cat > README.md << 'EOF'
# Грузавтотранс – платформа для грузоперевозок с картой и весом

