FROM dalgibbard/tsd_base:latest

ADD . /app
WORKDIR /app
RUN python3 -m pip install -U pip
RUN python3 -m pip install -r requirements.txt

RUN python manage.py collectstatic --noinput -c
