### 로컬에서 실행하기
```bash
# Gemfile.lock 생성
docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.6 bundle install

# Docker Image 빌드
docker-compose -f ./docker/docker-compose.build-image.yml build

# 127.0.0.1:4000 서버 생성
docker-compose -f ./docker/docker-compose.default.yml up
```