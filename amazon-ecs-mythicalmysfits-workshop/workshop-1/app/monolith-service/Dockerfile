FROM ubuntu:latest
RUN apt-get update -y
RUN apt-get install -y python-pip python-dev build-essential
RUN pip install --upgrade pip
COPY service/requirements.txt .
RUN pip install -r ./requirements.txt
COPY ./service /MythicalMysfitsService
WORKDIR /MythicalMysfitsService
EXPOSE 80
ENTRYPOINT ["python"]
CMD ["mythicalMysfitsService.py"]