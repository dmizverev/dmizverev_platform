FROM python:3-alpine

RUN mkdir /app
VOLUME /app

RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser
USER appuser

EXPOSE 8000

WORKDIR /app

CMD ["python", "-m" , "http.server", "8000"]