#<<<<<<< HEAD
FROM dalgibbard/tsd_base:latest
#=======
#FROM thespaghettidetective/web:base-1.6

#WORKDIR /app
#EXPOSE 3334
#>>>>>>> a9f605089c6df7d389f27dc0f5b0bc7d3c3403d5

ADD . /app
WORKDIR /app
RUN python3 -m pip install -U pip
RUN python3 -m pip install -r requirements.txt

RUN python manage.py collectstatic --noinput -c
