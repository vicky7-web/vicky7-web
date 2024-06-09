# taskmanager/models.py

from django.db import models
import uuid

class ScrapingJob(models.Model):
    job_id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, default='PENDING')

class ScrapingTask(models.Model):
    job = models.ForeignKey(ScrapingJob, related_name='tasks', on_delete=models.CASCADE)
    coin = models.CharField(max_length=50)
    status = models.CharField(max_length=20, default='PENDING')
    data = models.JSONField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
# taskmanager/serializers.py

from rest_framework import serializers
from .models import ScrapingJob, ScrapingTask

class ScrapingTaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = ScrapingTask
        fields = ['coin', 'status', 'data']

class ScrapingJobSerializer(serializers.ModelSerializer):
    tasks = ScrapingTaskSerializer(many=True)

    class Meta:
        model = ScrapingJob
        fields = ['job_id', 'created_at', 'status', 'tasks']
# taskmanager/coinmarketcap.py

import requests
from bs4 import BeautifulSoup

class CoinMarketCap:
    BASE_URL = 'https://coinmarketcap.com/currencies/'

    @staticmethod
    def get_coin_data(coin):
        url = f"{CoinMarketCap.BASE_URL}{coin}/"
        response = requests.get(url)
        if response.status_code != 200:
            return None

        soup = BeautifulSoup(response.content, 'html.parser')
        data = {}

        # Scraping logic here (replace with actual logic for each required field)
        data['price'] = soup.find('div', class_='priceValue___11gHJ').text.strip()
        # Continue extracting other fields...

        # Example:
        # data['market_cap'] = soup.find('div', {'id': 'market-cap'}).text.strip()
        # ...and so on

        return data

    @staticmethod
    def scrape(coin_list):
        results = {}
        for coin in coin_list:
            results[coin] = CoinMarketCap.get_coin_data(coin)
        return results
# project_name/celery.py

from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project_name.settings')

app = Celery('project_name')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
# taskmanager/tasks.py

from celery import shared_task
from .models import ScrapingJob, ScrapingTask
from .coinmarketcap import CoinMarketCap

@shared_task
def start_scraping(job_id, coins):
    job = ScrapingJob.objects.get(job_id=job_id)
    for coin in coins:
        task = ScrapingTask.objects.create(job=job, coin=coin)
        data = CoinMarketCap.get_coin_data(coin)
        if data:
            task.data = data
            task.status = 'COMPLETED'
        else:
            task.status = 'FAILED'
        task.save()
    job.status = 'COMPLETED'
    job.save()
# taskmanager/views.py

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import ScrapingJob, ScrapingTask
from .serializers import ScrapingJobSerializer
from .tasks import start_scraping
import uuid

class StartScrapingView(APIView):
    def post(self, request):
        coins = request.data
        if not isinstance(coins, list) or not all(isinstance(coin, str) for coin in coins):
            return Response({'error': 'Invalid input. Expected a list of strings.'}, status=status.HTTP_400_BAD_REQUEST)
        
        job = ScrapingJob.objects.create()
        start_scraping.delay(str(job.job_id), coins)
        return Response({'job_id': job.job_id}, status=status.HTTP_202_ACCEPTED)

class ScrapingStatusView(APIView):
    def get(self, request, job_id):
        try:
            job = ScrapingJob.objects.get(job_id=job_id)
        except ScrapingJob.DoesNotExist:
            return Response({'error': 'Job not found.'}, status=status.HTTP_404_NOT_FOUND)
        
        serializer = ScrapingJobSerializer(job)
        return Response(serializer.data, status=status.HTTP_200_OK)
# taskmanager/urls.py

from django.urls import path
from .views import StartScrapingView, ScrapingStatusView

urlpatterns = [
    path('start_scraping/', StartScrapingView.as_view(), name='start_scraping'),
    path('scraping_status/<uuid:job_id>/', ScrapingStatusView.as_view(), name='scraping_status'),
]
# project_name/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/taskmanager/', include('taskmanager.urls')),
]
pip install django djangorestframework celery beautifulsoup4 requests
python manage.py makemigrations
python manage.py migrate
celery -A project_name worker --loglevel=info
