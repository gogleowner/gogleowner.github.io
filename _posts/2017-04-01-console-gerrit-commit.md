---
layout: post
title: "git-review로 콘솔환경에서 gerrit에 push하기"
date: 2017-04-01 06:37:00 +0900
categories: git
---

# git-review로 콘솔환경에서 gerrit에 push하기

2015년 초반에 작성한 글이다. 당시 회사에서 [gerrit](https://www.gerritcodereview.com)을 활용하여 작업 내용 코드리뷰, 머지를 진행하고 있었다. eclipse에서는 origin 을 gerrit 으로 변경하여 gerrit 에 푸쉬할 수 있었는데 개발 IDE로 IntelliJ를 처음 지급받았을때 gerrit 버전이 낮아서 gerrit 플러그인을 사용하지 못하여.. `git-review` 를 찾아내어, 정리했었다.

> 이글은.. 특이하게도 존댓말이다.

## 계기
- eclipse에서는 Egit으로 git 작업을 했었는데, IntelliJ에서는 어떻게 할지 방법을 찾다가, 콘솔을 이용하는 방법을 찾았습니다.

## 왜 콘솔인가?
- IntelliJ 플러그인 중 gerrit 플러그인이 있어서 설치했는데, gerrit 버전 호환 문제로 인해 gerrit 플러그인의 사용이 불가능했습니다.
	- IntelliJ git 플러그인은 *v2.6 이상부터 지원합니다... 회사에서 쓰던 gerrit 현재 v2.5.2이라서 이렇게 했었습니다.*
- 그래서 콘솔에서 **git-review**를 활용해서 gerrit 연동을 했습니다.


## git-review 이용방법
- [git-review](http://docs.openstack.org/infra/git-review/)는 콘솔 명령어로 gerrit에 push할 수 있게 해주는 툴(오픈소스)입니다.
- git-review를 설치, 세팅 후 gerrit에 push한 것 위주로 설명드리겠습니다.(위의 링크에 더 자세한 내용이 나와있습니다.)

### 1. 설치
- 홈페이지에는 ```pip```로 설치하는 방법이 나와 있습니다.
- OSX 환경에서는 [brew](http://www.brew.sh/)로도 설치할 수 있습니다.

	```
	$ brew install git-review
	```


### 2. repository directory에서 .gitreview 파일 생성, 셋업
- ```.gitreview``` 파일에는 gerrit 서버정보, project, branch 등을 작성합니다.
- 아래 코드는 제가 작성한 .gitreview 파일입니다.

	```
	[gerrit]
	host=repos.tmonc.net
	port=2222
	project={리파지토리 이름}
	defaultbranch=master
	```

- 그 후, ```$ git review -s``` 으로 git-review를 셋업합니다.

### 3. git commit
- 작업한 내용을 commit 합니다. 저의 경우 ```$ git commit``` 을 하면, Sublime Text로 커밋메시지를 입력하도록 해놓았는데요. 
- 커밋메시지에 **changeId** 가 있으면 gerrit에 push할 준비가 된 것입니다. 바로 5번 과정으로 넘어갑니다.
![commit-msg](https://i.postimg.cc/Jhgq5WPM/commit-msg.png)
- changeId가 없으면 아래 과정을 진행하고 5번 과정으로 넘어갑니다.
- 아래 과정을 진행했는데도 changeId 가 나오지 않는다면, ```$ git commit --amend``` 명령어를 한번 더 입력하면 changeId 가 나올 것입니다.

### 4. gerrit 서버에서 commit-msg hook 복사*(changeId 가 없을 경우)*
- 기존에 eclipse의 egit으로 작업한 프로젝트면 commit-msg hook 세팅되있어서 바로 push가 될 것입니다.
- repository에서 처음으로 clone을 받아, 최초 push 작업을 진행하는 경우라면, commit-msg hook 세팅이 되어있지 않아서 다음과 같은 메시지가 나옵니다.

	```
	$ git review
	Traceback (most recent call last):
	  File "/usr/local/Cellar/git-review/1.24/libexec/bin/git-review", line 10, in <module>
	    sys.exit(main())
	  File "/usr/local/Cellar/git-review/1.24/libexec/lib/python2.7/site-packages/git_review/cmd.py", line 1202, in main
	    set_hooks_commit_msg(remote, hook_file)
	  File "/usr/local/Cellar/git-review/1.24/libexec/lib/python2.7/site-packages/git_review/cmd.py", line 283, in set_hooks_commit_msg
	    run_command_exc(CannotInstallHook, *cmd)
	  File "/usr/local/Cellar/git-review/1.24/libexec/lib/python2.7/site-packages/git_review/cmd.py", line 152, in run_command_exc
	    raise klazz(rc, output, argv, env)
	git_review.cmd.CannotInstallHook: Problems encountered installing commit-msg hook
	The following command failed with exit code 1
	    "scp -P2222 seunghyolee@repos.tmonc.net:hooks/commit-msg .git/hooks/commit-msg"
	-----------------------
	lost connection
	-----------------------
	```

- 이를 해결하기 위해 gerrit 서버에서 commit-msg hook을 복사합니다.

	```
	$ scp -p -P 2222 {gerrit서버url}:hooks/commit-msg .git/hooks/
	commit-msg                                                                100% 4270     4.2KB/s   00:00
	lost connection
	```

- commit한 상태라면, ```$ git commit --amend``` 을 이용해 commit message에 changeId 가 있는 것을 확인합니다.
- 이제 gerrit에 push 할 준비가 되었습니다.
- *왜 이 과정을 진행했는지, changeId는 무엇인지에 대해 하단에 적어놓았습니다.*


### 5. gerrit에 push(changeId 가 있을 경우)
- push를 수행합니다.

	```
	$ git review
	remote: Resolving deltas: 100% (104/104)
	remote: Processing changes: new: 1, refs: 1, done
	remote:
	remote: New Changes:
	remote:   {gerrit서버url}/33752
	remote:
	To ssh://{gerrit서버url}/{리포지토리명}
	 * [new branch]      HEAD -> refs/publish/master
	```

- 위와 같은 메시지가 나오면 성공입니다.
- gerrit사이트에 들어가면 푸쉬 내역이 나옵니다.
![gerrit_review_shot](https://i.postimg.cc/mrqvzJ9F/gerrit-review-shot.png)
- 여기까지 git-review 프로세스의 흐름이었습니다.


## 조사한 자료
### changeId란?
- gerrit에서 change내역을 알아보기 위한 id값입니다.
- changeId를 보고 이 change가 이전에 올려졌던 change에 대한 patchset인지, 새로 생성되어야할 change인지 식별합니다.
![change-id](https://i.postimg.cc/R083DSkp/change-id.png)


### gerrit 서버의 [commit-msg hook](https://gerrit.googlecode.com/svn/documentation/2.0/cmd-hook-commit-msg.html)란?

- 위와 같은 이유로 gerrit에서 change 관리를 위해 commit message에 changeId를 꼭 삽입해야 합니다.
	
	> commit-msg Hook : Edit commit messages to insert a Change-Id tag. (from google gerrit documentation)
- commit 할때마다 나오는 changeId를 세팅하기 위해, gerrit 서버에서 commit-msg hook을 복사합니다.

	```
	$ cat .git/hooks/commit-msg
	#!/bin/sh
	# From Gerrit Code Review 2.5.2
	# Part of Gerrit Code Review (http://code.google.com/p/gerrit/)
	......(길어서 생략합니다.)
	```

- 유일한 id 값의 changeId 를 만들어내는 스크립트 파일인 것 같습니다.
- eclipse의 egit에서 gerrit으로 연동하여 commit을 할때, commit-msg을 통해 changeId를 만드는 것 같습니다.


### (번외) changeId에 아무 값을 넣은다면?
- eclipse로 했을때랑 다른 점이 changeId 가 없었다는 점이라는 점을 깨달아서, commit message에 changeId에 값을 넣어봤습니다.

	```
	$ git review
	remote: Resolving deltas: 100% (23/23)
	remote: Processing changes: refs: 1, done
	To ssh://{gerrit서버url}/{리포지토리명}
	 ! [remote rejected] HEAD -> refs/publish/master (invalid Change-Id)
	error: failed to push some refs to 'ssh://seunghyolee@{gerrit서버url}/{리포지토리명}'
	```

- changeId가 Invalid해서 push 못한다는 에러가 발생합니다.

## References
- [commit-msg hook](https://gerrit.googlecode.com/svn/documentation/2.0/cmd-hook-commit-msg.html)
- [changeId 설명](http://lapan.tistory.com/63)
- [git-review를 개발하고있는 openstack홈페이지](http://docs.openstack.org/infra/git-review/)
- [git-review Workflow 간단한 것](https://wiki.opendaylight.org/view/Git-review_Workflow)
- [git-review Workflow 자세한 것](https://www.mediawiki.org/wiki/Gerrit/git-review#Submitting_changes_with_git-review)
- [intellij plugins](https://github.com/uwolfer/gerrit-intellij-plugin)
