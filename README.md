image: docker:latest
workflow:
  rules:
  - if: $CI_COMMIT_TAG  
    variables:
      COMMIT_ENV: "prod"
  - if: $CI_COMMIT_REF_NAME =~ /^test/
    variables:
      COMMIT_ENV: "test"
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    variables:
      COMMIT_ENV: "dev"
  - when: never
  
stages:
  - test
  - build
  - deploy


include:
  - project: 'cicd/test'
    ref: main
    file: '.gitlab-ci.yml'


build_app:
  stage: build
  image: gitlab.local.ru:5050/cicd/build/docker:19.03
  services:
    - docker:19.03-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  variables:
    APP: s-server-sync
    IMAGE_NAME: ${CI_REGISTRY_IMAGE}:$APP-${COMMIT_ENV}_${CI_COMMIT_SHORT_SHA}
  script:
    - envsubst < Dockerfile.tmpl > Dockerfile
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
    - image_digest=$(docker inspect --format='{{index .RepoDigests 0}}' "$IMAGE_NAME")
    - echo "image_digest=${image_digest}" >> image.digest
  tags: 
    - ecs-gitlab-runner-docker
  artifacts:
    reports:
      dotenv: image.digest


Deploy app:
  stage: deploy   
  trigger:     
    project: cicd/deploy   
    branch: main
    strategy: depend
  variables: 
    env: ${COMMIT_ENV}
    app_name: app_1
    group_name: "group_name_1"
    image_digest: ${image_digest}
