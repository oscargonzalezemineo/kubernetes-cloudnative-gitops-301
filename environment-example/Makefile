frontend-generate-app:
# 	docker compose run --rm frontend bash -c "ng new mi-app --defaults --skip-git" 
	docker compose run --rm frontend bash -c "chown -R 1000:1000 /app"

api-init:
# 	docker compose run --rm api bash -c "npm init"
	docker compose run --rm api bash -c "chown -R 1000:1000 /app"

angular-cli:
	docker compose run --rm -it frontend bash

api-cli:
	docker compose run --rm -it api bash

install: 
	docker compose run --rm frontend bash -c "npm i"
	docker compose run --rm api bash -c "npm i"




start:
	docker compose up -d

logs:
	docker compose logs -f



build-prod:
	docker build -t angular-prod ./src/frontend/mi-app
	docker build -t api-prod ./src/api

