FROM python:alpine
RUN apk add --no-cache tini
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
WORKDIR /app
COPY . /app
EXPOSE 80
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["python", "./app.py"]
