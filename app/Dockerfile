FROM python:3.6-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd --create-home --uid 10001 app

RUN chown -R app:app /app

USER app

EXPOSE 8080

CMD ["python", "app.py"]
