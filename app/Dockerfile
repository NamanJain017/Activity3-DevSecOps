FROM nginx/nginx-unprivileged:alpine3.24

COPY index.html /usr/share/nginx/html/index.html

USER 101

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 CMD wget -q --spider http://127.0.0.1:8080/ || exit 1
