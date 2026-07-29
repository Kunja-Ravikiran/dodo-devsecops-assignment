FROM python:3.12-alpine
WORKDIR /app
RUN adduser -D -u 1000 appuser
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
