FLAG ?= GCDSTE{d1sc0rd_gh0st_1n_th3_sh3ll_4nd_th3_l1ar}
COMPOSE = docker compose --env-file secrets.env -f infra/docker-compose.yml

.PHONY: secrets step2 image up dev down test
secrets:
	python tools/gen_secrets.py --flag '$(FLAG)'
step2:
	python step2-stego/build_step2.py
image:
	bash step4-docker/build_image.sh
up:
	$(COMPOSE) up -d --build
dev:
	$(COMPOSE) -f infra/docker-compose.dev.yml up -d --build
down:
	$(COMPOSE) down
test:
	python -m pytest -q tests
