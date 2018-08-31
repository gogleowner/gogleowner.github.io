---
layout: post
title: "git 복구방법 모음"
date: 2018-06-12 15:10:00 +0900
categories: seminar
---


# git 복구

- 특정 시점으로 되돌리고 싶은 경우
	- ```reset``` 명령어 사용
		- 명령어
			- ```git reset HEAD^``` : 최종 커밋을 취소, 워킹트리 보존
			- ```git reset HEAD~n``` : 마지막 n개의 커밋을 취소, 워킹트리 보존
			- ```git reset --hard HEAD~n``` : 마지막 n개의 커밋을 취소, index 및 워킹트리 모두 원복
			- ```git reset --hard {특정시점의 커밋id}``` : 이 시점 까지의 index 및 워킹트리 모두 원복
		- ```reset``` 옵션
			- ```--soft``` : index 보존
			- ```--mixed``` : index 취소, 워킹트리만 보존
			- ```--hard``` : index 취소, 워킹트리 취소

- 특점시점으로 돌리는데, 푸쉬까지 해버린 경우
	- ```git reset --hard {특정시점의 커밋id}```
		- 로컬 브랜치의 워킹트리를 원복하고,
	- ```git push {리모트} +{브랜치}```
		- ```+``` 옵션을 주지 않으면 git에서 reject 함.

- 푸쉬까지 해버렸는데, 커밋메시지의 정정이 필요한 경우
	- ```git commit --amend -m "새로운 커밋 메시지"```
		- 로컬 브랜치의 커밋메시지 정정
	- ```git push {리모트} {브랜치} -f``` or ```git push {리모트} {브랜치} --force```
		- force-pushing 은 로컬의 내용을 remote에 overwrite 하게 된다.