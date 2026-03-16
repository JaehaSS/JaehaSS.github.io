---
layout: post
title: "AWS ECR 비용 최적화 해보기(Python 스크립트를 곁들인)"
date: 2025-01-19
description: "AWS ECR의 스토리지 비용을 최적화하기 위해 Python과 Lambda를 활용한 이미지 자동 정리 스크립트를 작성하는 방법을 공유합니다."
tags: [Python]
tistory_id: 55
---

# AWS ECR 비용 최적화 해보기(Python 스크립트를 곁들인)

## ECR 이란?

Amazon Elastic Container Registry(Amazon ECR)는 어디에서나 컨테이너 소프트웨어를 손쉽게 저장, 공유 및 배포할 수 있는 완전관리형 컨테이너 레지스트리입니다. Docker Hub 같은 개념이라고 생각하면 편합니다.

## ECR 과금 방식

ECR의 과금 방식에는 크게 데이터 송수신 비용과 스토리지 비용이 존재합니다.

### 1. 데이터 송수신

![데이터 송수신 비용](https://blog.kakaocdn.net/dna/AOE41/btsLRINtw1d/AAAAAAAAAAAAAAAAAAAAANins_gn3QnOZv9XhMsF2SA3xc0YNqw4jJVxoQiQpzna/img.png)

### 2. 스토리지 비용

![스토리지 비용](https://blog.kakaocdn.net/dna/b2bwKO/btsLT3vo5JS/AAAAAAAAAAAAAAAAAAAAAPrHFIerKMV7hiEuiLHZKQ33QmEh_EccVXprHEbrZUj7/img.png)

## ECR 비용 최적화

데이터 송수신 비용과 스토리지 비용 중에 스토리지 비용 최적화에 대해서 글을 작성합니다.

현재 회사에서는 운영 서버에 배포할 때 GitHub Action + CI/CD 활용한 AWS ECS에 자동 배포를 하고 있습니다.

자동 배포를 할 때마다 ECR의 레포지토리에는 이미지가 점점 쌓입니다.

![ECR 레포지토리 이미지](https://blog.kakaocdn.net/dna/N3ZJ7/btsLSBNCKFl/AAAAAAAAAAAAAAAAAAAAAMB7-BnYGRUGUEe8MUK7HK1qqFjgWq-ZevS2CW_d5KyO/img.png)

회사에는 하나의 Product가 아닌 여러 개의 Product가 존재할 수도 있습니다. 그럼 수많은 이미지가 존재할 수도 있겠네요. 이렇게 사용 안하는 이미지들을 제거해야 스토리지 비용 최적화가 가능합니다.

![AWS ECR 콘솔](https://blog.kakaocdn.net/dna/bMMJoT/btsLTwxXUNx/AAAAAAAAAAAAAAAAAAAAAC-JN4Jh99opxHyHwU5KGpbKwJ3B3TvqSv9cUTkgYTXB/img.png)

AWS ECR 콘솔에 들어가 수동으로 삭제할 수 있습니다. 하지만 개발자라면 자동으로 해야하지 않을까요?

### ECR 이미지 제거 스크립트 - Python과 AWS Lambda을 곁들인

```python
import boto3
import requests
import json
from typing import List

ecr_client = boto3.client(
    "ecr",
    aws_access_key_id="AWS Access Key",
    aws_secret_access_key="AWS Secret Key",
)


# 모든 ECR 레포지토리 리스트 가져오기
def get_ecr_repositories():
    repositories = []
    response = ecr_client.describe_repositories()
    repositories.extend(response["repositories"])

    while "nextToken" in response:
        response = ecr_client.describe_repositories(nextToken=response["nextToken"])
        repositories.extend(response["repositories"])

    return repositories


# 특정 레포지토리의 이미지 리스트 가져오기
def get_ecr_images(repository_name):
    images = []
    response = ecr_client.list_images(repositoryName=repository_name)
    images.extend(response["imageIds"])

    while "nextToken" in response:
        next_token = response["nextToken"]
        response = ecr_client.list_images(
            repositoryName=repository_name, nextToken=next_token
        )
        images.extend(response["imageIds"])

    return images


# 이미지 삭제
def delete_ecr_images(repository_name, images):
    response = ecr_client.batch_delete_image(
        repositoryName=repository_name, imageIds=images
    )
    return response


# List Chunk
def list_chunk(lst, n):
    return [lst[i : i + n] for i in range(0, len(lst), n)]


# 이미지 정렬
def sort_ecr_images(repository_name, images: List):
    data = []

    if len(images) > 100:
        chunk_data = list_chunk(images, 99)
    else:
        chunk_data = [images]

    for i in chunk_data:
        res = ecr_client.describe_images(repositoryName=repository_name, imageIds=i)
        res = [
            [i["imageDigest"], i["imagePushedAt"], i["imageSizeInBytes"]]
            for i in res["imageDetails"]
        ]
        data.extend(res)

    # 0  이미지 제거
    remove_images_ids = []
    for index, i in enumerate(data):
        if i[2] < 2000:
            remove_images_ids.append({"imageDigest": i[0]})
            del data[index]

    # 정렬 - 최신 순
    data.sort(key=lambda x: x[1], reverse=True)

    data = [{"imageDigest": i[0]} for i in data]

    return data


def slack_message(message):
    url = "Slack URL"

    payload = {"text": message}

    requests.post(
        url,
        data=json.dumps(payload),
        headers={"Content-Type": "application/json"},
    )


def main():
    # 레포지토리 리스트 가져오기
    repositories = get_ecr_repositories()

    for repo in repositories:
        repo_name = repo["repositoryName"]
        images = get_ecr_images(repo_name)

        if len(images) > 5:
            # 이미지 정렬 후 삭제
            data = sort_ecr_images(repo_name, images)

            if len(data) <= 5:
                continue

            delete_ecr_images(repo_name, data[5:])

            delete_ecr_images_count = len(data) - 5

            slack_message(
                f":amongusdevil: ECR Repo Image 삭제 완료:amongusdevil: \n"
                f"삭제된 이미지 Repo : {repo_name} \n"
                f"삭제된 이미지 수 : {delete_ecr_images_count} 개 \n"
            )


def lambda_handler(event, context):
    main()

    return None
```

### 추가 작업

AWS Lambda를 만들었지만, 자동화하기 위해서는 해당 Lambda를 스케쥴러를 해야합니다.

AWS에서 제공해주는 EventBridge 서비스를 사용하면 편하게 스케쥴링이 가능합니다! 이건 읽으시는 독자분이 번외로 해보시면 재밌을 거 같네요.
