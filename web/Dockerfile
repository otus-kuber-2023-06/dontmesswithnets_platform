FROM nginx
RUN mkdir -p /app && chown -R nginx:nginx /app
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8000
CMD ["nginx", "-g", "daemon off;"]