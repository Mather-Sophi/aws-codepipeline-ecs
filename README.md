## aws-codepipeline-ecs
Creates a pipeline that builds a container, pushes it to ECR and deploys the container to an ECS service

## v1.6 Note
The account that owns the guthub token must have admin access on the repo in order to generate a github webhook 

## v1.7 Note
If `use_docker_credentials` is set to `true`, the environment variables `DOCKERHUB_USER` and `DOCKERHUB_PASS` are exposed via codebild

You can add these 2 lines to the beginning of your `build` phase commands in `buildspec.yml` to login to Dockerhub

```yml
  build:
    commands:
      - echo "Logging into Dockerhub..."
      - docker login -u ${DOCKERHUB_USER} -p ${DOCKERHUB_PASS}
      ...
      ...
```

## Usage

```hcl
module "ecs_pipeline" {
  source = "github.com/globeandmail/aws-codepipeline-ecs?ref=1.7"

  name               = "app-name"
  ecr_name           = "ecr-repo-name"
  ecs_cluster_name   = "cluster-name"
  ecs_service_name   = "service-name"
  github_repo_owner  = "github-account-name"
  github_repo_name   = "github-repo-name"
  github_oauth_token = data.aws_ssm_parameter.github_token.value
  tags = {
    Environment = var.environment
  }
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| name | The name associated with the pipeline and assoicated resources. ie: app-name | string | n/a | yes |
| ecr\_name | The name of the ECR repo | string | n/a | yes |
| ecs\_cluster\_name | The name of the ECS cluster | string | n/a | yes |
| ecs\_service\_name | The name of the ECS service | string | n/a | yes |
| ecs\_service\_name | The name of the ECS service | string | n/a | yes |
| github\_repo\_owner | The owner of the GitHub repo | string | n/a | yes |
| task\_execution\_role | The name of the task execution role | string | `"ecsTaskExecutionRole"` | no |
| github\_oauth\_token | GitHub oauth token | string | n/a | yes |
| codebuild\_image | The codebuild image to use | string | `"null"` | no |
| buildspec | build spec file other than buildspec.yml | string | `"buildspec.yml"` | no |
| ecs\_artifact\_filename | The name of the ECS deploy artifact | string | `"null"` | no |
| github\_branch\_name | The git branch name to use for the codebuild project | string | `"master"` | no |
| use\_docker\_credentials | \(Optional\) Use dockerhub credentals stored in parameter store | bool | false | no |
| tags | A mapping of tags to assign to the resource | map | `{}` | no |

## Outputs

| Name | Description |
|------|-------------|
| artifact\_bucket\_arn |  |
| artifact\_bucket\_id |  |
| codebuild\_project\_arn |  |
| codebuild\_project\_id |  |
| codepipeline\_arn |  |
| codepipeline\_id |  |

## Builspec example

```yml
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: "ecr-repo-name"

phases:
  install:
    runtime-versions:
      docker: 18
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
      - AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
      - REPOSITORY_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - printf '[{"name":"%s","imageUri":"%s"}]' $IMAGE_REPO_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files: imagedefinitions.json
```
