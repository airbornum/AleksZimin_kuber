# Скачиваем официальный образ nginx. Помечаем его как original_image
FROM nginx:1.17.3-alpine as original_image
# Далее создаем пустой образ
FROM scratch as nginx
# Копируем все содержимое original_image в пустой образ. Таким образом мы удаляем все метаданные, которые были в официальном образе nginx (volumes, expose ports, CMD)
COPY --chown=1001:1001 --from=original_image / /
RUN  mkdir -p /app && \
      addgroup -S appgroup -g 1001  && \
      adduser -S appuser -D -u 1001 -G appgroup -g appgroup
USER appuser
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8000
CMD ["nginx", "-g", "daemon off;"]
