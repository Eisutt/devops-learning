# devops-learning

Обучение DevOps / L2 поддержке инфраструктуры: 12 недель, ~12 часов в неделю.
Практика на VPS в Yandex Cloud, каждый шаг задокументирован.

## План

Статус: ⬜ не начато · 🔄 в процессе · ✅ готово

| Нед | Тема | Практика на VPS | Статус |
|-----|------|-----------------|--------|
| 1 | Linux вглубь: процессы, сигналы, systemd, журналы, диски, LVM | харднинг сервера, свой systemd-сервис | 🔄 |
| 2 | Сети: TCP/IP, подсети, DNS, HTTP, TLS | `dig`, `curl -v`, `tcpdump`, `ss` | ⬜ |
| 3 | Nginx: reverse proxy, TLS (Let's Encrypt), логи | свой домен или nip.io, HTTPS | ⬜ |
| 4 | Bash, cron, Git | скрипты бэкапа и мониторинга | ⬜ |
| 5 | Траблшутинг и PostgreSQL | инциденты-симуляции, бэкап и восстановление БД | ⬜ |
| 6 | Docker | Dockerfile, многослойные образы, сети, volumes | ⬜ |
| 7 | Docker Compose и CI/CD (GitHub Actions) | автодеплой приложения на VPS | ⬜ |
| 8 | Мониторинг: Prometheus, Grafana, Loki, алерты | дашборды, алерт в Telegram | ⬜ |
| 9 | Ansible | весь сервер настраивается плейбуком | ⬜ |
| 10 | Terraform и Yandex Cloud | инфраструктура кодом | ⬜ |
| 11–12 | Kubernetes (k3s) и финальный проект | деплой в кластер | ⬜ |

## Финальный проект

Приложение в Docker: инфраструктура в Yandex Cloud поднимается Terraform'ом,
сервер настраивается Ansible'ом, деплой идёт через CI/CD, подключены мониторинг и алерты.

## Заметки

### Неделя 1: Linux
- [01. Подготовка: репозиторий и сервер](week-01-linux/01-server-setup.md)
- [02. Харднинг: sudo и SSH](week-01-linux/02-sudo-ssh-hardening.md)
- [03. Харднинг: firewall (ufw) и fail2ban](week-01-linux/03-firewall-fail2ban.md)
