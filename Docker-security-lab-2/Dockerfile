FROM python:3.15-slim
WORKDIR /app2
RUN pip uninstall -y pip
COPY app/app.py /app2
CMD [ "python","app.py" ]