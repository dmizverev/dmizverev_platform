FROM nginx:1.17.1-alpine

COPY nginx.conf /etc/nginx/
COPY site.conf /etc/nginx/conf.d/
EXPOSE 8000
CMD ["nginx", "-g", "daemon off;"]


