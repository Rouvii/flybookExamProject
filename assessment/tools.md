# Værktøjer

For hvert værktøj: version, præcis kommando, indstillinger/regelsæt, tuning, antal fund før/efter frasortering af falske positiver og begrundelse.

## Miljø

| Komponent | Version / detalje |
|---|---|
| Container-runtime | Rancher Desktop med dockerd (moby), Docker Engine 29.5.3 |
| Docker-socket | `unix://$HOME/.rd/docker.sock` (sat via `DOCKER_HOST`) |
| Web-image | `flybookexamproject-web:latest` (bygget fra `Dockerfile`, base `php:7.3-apache`) |
| DB-image | `mysql:8.0` |
| Applikation | http://localhost:8000 |

Opstart:

    export DOCKER_HOST=unix://$HOME/.rd/docker.sock
    docker-compose build && docker-compose up -d

## Sammenligning af værktøjer

_Udfyldes i fase 2._
