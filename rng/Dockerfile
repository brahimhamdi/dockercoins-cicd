FROM python:3.11-alpine
WORKDIR /app
RUN pip install --no-cache-dir flask
COPY rng.py .
RUN adduser -D rng && chown -R rng:rng /app
USER rng
EXPOSE 80
CMD ["python", "rng.py"]
