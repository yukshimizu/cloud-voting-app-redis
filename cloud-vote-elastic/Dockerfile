FROM tiangolo/uwsgi-nginx-flask:python3.8
RUN pip install redis blinker elastic-apm prometheus-client
ADD /cloud-vote /app
