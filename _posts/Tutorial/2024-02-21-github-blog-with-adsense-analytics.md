---
title: "Github 블로그 개설 및 Google Adsense 와 Google Analytics 적용"
comments: true
toc: true
math: true
mermaid: true
date: 2024-02-20
tags: [ Google Adsense , Google Analytics, Jekyll, Github Pages]
---
안녕하세요! 그동안 개발 기술 블로그에 정착하지 못 하고 있었는데요 

개발 기술 블로그를 고민하던 도중 Github Pages 를 이용한 블로그 만드는 법을 발견하고 진행하게 되었는데, 오늘은 이 경험을 토대로
Github 블로그 만드는 방법 및 Google Adsense 및 Google Analytics 연결에 대해 작성해보겠습니다. 🙇🏻‍♀️ 

# Github Pages 를 통한 블로그 제작
---

[GitHub Pages](https://docs.github.com/ko/pages/quickstart) 는 GitHub를 통해 호스트되고 게시되는 퍼블릭 웹 페이지입니다.
Github 저장소에 저장된 정적 웹문서를 Github 에서 무료로 웹에서 볼 수 있도록 호스팅 하는 기능을 제공합니다. 

## Github 블로그 만들기

가장 간단한 방법으로 Github 블로그를 제작해보려고 합니다.

마음에 드는 Jekyll Theme 를 찾아 Fork 해줍니다. <BR>
그 후, Fork 받은 레파지토리 이름을 `{github 아이디}.github.io` 로 변경해줍니다. <BR>
github 레파지토리를 `{github 아이디}.github.io` 이름으로 생성 후 원하는 테마 파일을 설치 및 붙여 넣어 실행도 가능하지만, 저는 왜인지 파일 간 충돌로 인해 간단하게 Fork 해주는 방법을 선택했습니다. Fork 로 진행 시 ui, google analytics, adsense 등 커스터마이징 하기 수월하다는 장점이 있습니다. 다만, Fork 로 진행 시 Commit 내역이 깔끔하지 못하다는 단점이 있습니다.

저는 `Chirpy` 테마를 Fork 해주었습니다. 

### Ruby 설치하기

Jekyll 은 Ruby 기반 정적 웹사이트 생성기이기 때문에, Ruby 설치가 필수적입니다.



Ruby 버전 관리 패키지인 [rbenv](https://github.com/rbenv/rbenv) 를 설치해봅시다. rbenv는 ruby의 가상환경을 구축하고 여러 개의 버전 설치를 가능하게 합니다. mac OS 를 기준으로 작성 하겠습니다. 

macOS 용 패키지 관리자인 homebrew 를 이용해 rbenv 를 설치해줍니다.
```
brew install rbenv
```

이후, 다운 받고자 하는 rbenv 버전을 선택 후 설치합니다.
``` 
# Check the versions you can install
rbenv install -l

# Install rbenv version
rbenv install ${다운받고 싶은 버전}
```
![img](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/8d83f706-af02-4087-9333-fc8ae16984cc)

현재 ruby version 을 확인 후, 설치한 버전으로 변경해줍니다.

```
# Check the current version
rbenv versions

# Change version
rbenv global ${설치한 버전}
```
zsh 에서 rbenv를 실행할 수 있게 쉘 환경변수를 수정하겠습니다. zsh 설정 파일을 열어, rbenv PATH 추가 설정 및 저장을 해줍니다.
```
# Open zsh setting file 
vi ~/.zshrc 

# Add path
export PATH=${Home}/.rbenv/bin:$PATH && \
eval "$(rbenv init -)" 

# Save zsh setting file 
source ~/.zshrc 
```

bash 를 사용하는 경우, ./bash_profile 에 작성하셔도 무방합니다. 다만, MacOS에서는 zsh로의 전환을 장려하고 있습니다.

### Jkeyll 설치하기

GithubPage 제작 프로그램으로 가장 많이 사용되는 Jekyll은 GitHub Pages 및 간소화된 빌드 프로세스를 기본적으로 지원하는 정적 사이트 생성기입니다. <BR> Jekyll은 Markdown 및 HTML 파일을 가져와 선택한 레이아웃에 따라 정적 웹 사이트를 만듭니다.

```
# Install jekyll
gem install jekyll

# Check if it's installed normally
jekyll -v 
```

저는 설치하는 도중 gem 권한 에러가 발생했습니다.
![img](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/e9577860-2483-4cd5-ada5-2412acf48d1f)
아직 Ruby 개발용으로 적절하게 구성되지 않은 Mac에 gem을 설치하려고 하면 해당 오류가 발생합니다. <BR> 
모든 Mac에 Ruby를 사전 설치 되어 있습니다. 따라서 시스템 Ruby를 사용하여 gem을 설치하려고 하면 gem이 설치되는 기본 위치는 디렉터리인 `/Library/Ruby/Gems/2.6.0` 입니다. 
그러나 해당 디렉토리는 수정될 수 없고, Apple 은 해당 디렉토리에 쓸 수 있는 권한을 부여하지 않으므로 다음과 같은 오류가 발생합니다.
다시 말해, 앞서 진행한 PATH 환경 설정이 적용이 제대로 진행되지 않아 기본 디렉토리의 Ruby를 사용하며 발생한 오류라고 볼 수 있습니다.

sudo 를 붙여줘도 해결이 되지만, `sudo gem` 으로 설치하거나 시스템 파일 및 디렉터리의 권한을 변경하는 건 수행하려는 작업이 명확하더라도 권장하지 않습니다. <BR> 
즉, 다음과 같은 오류 발생 시, 위의 과정을 다시 진행한 후 설치하면 해결할 수 있습니다. 이때, ruby 3.1.3 이상으로 진행하셔야 합니다.

### Local Test

마지막으로 로컬 환경에서 블로그 파일을 실행해서 잘 작동하는지 테스트 해보겠습니다.

Bundler 는 Ruby 프로젝트에 필요한 Gem 들의 올바른 버전을 추적하고 설치해서 일관된 환경을 제공합니다. <BR> 
따라서 Bundler는 Jekyll 을 시스템 레벨이나 사용자 레벨에 설치하고 싶지 않을때 특히 유용합니다. 

이제 [Bundler](https://bundler.io/) 를 설치해줍니다. <BR> 
이때, Fork 받은 Source 에 이미 dependencies 관련 Gemfile 이 존재 하므로 다음과 같은 명령어만 입력해주면 됩니다. <BR> 
없을 시, 프로젝트 루트의 Gemfile에 dependencies을 지정해야 합니다.

```
bundle install
```

이제 프로젝트 폴더에 설치된 Jekyll 버전을 Bundler가 실행할 수 있도록 bundle exec 명령어를 통해 웹사이트를 서버에 올려줍니다.
```
bundle exec jekyll serve
```
저는 명령어 실행 도중 다음과 같은 에러가 발생했습니다.
![img](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/0fe0a576-2b5c-4d68-8db6-d645536155b2)
이는 [homebrew를 통해 설치되는 최신 버전의 Ruby 에서 기본적으로 webrick가 포함되어 있지 않아 생기는 이슈](https://github.com/github/pages-gem/issues/752)입니다. 따라서 Ruby 2.7 이하 버전을 사용하거나 `webrick` 을 수동으로 설치해줍니다.
```
bundle add webrick
```
또 `undefined method 'untaint' for an instance of String (NoMethodError)` 와 같은 에러 발생 시, 이는 lock 파일이 이전 버전 Ruby에 종속하기 때문이므로 `Gemfile.lock` 파일을 삭제해주면 됩니다.
![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/e96b8c24-90ec-460a-9676-dc70429593be)

정상적으로 실행이 완료되면 `http://127.0.0.1:4000` 를 통해 사이트 접속이 가능합니다.

Fork 를 통해 블로그를 생성하지 않았거나, Local Test 과정에서 오류가 발생했다면 [Jekyll 튜토리얼](https://jekyllrb.com/tutorials/using-jekyll-with-bundler/
)을 참고하시면 좋을 것 같습니다.

### 구조 살펴보기
파일 구조를 대략적으로 살펴보겠습니다.

> _data : 언어에 따른 단어 설정 및 폰트와 라이브러리, 저자 연락망과 정보, 포스트 공유하기 등의 구성이 담겨있는 디렉토리입니다. <BR>
> _include : 포스트 관련 UI, header, footer, 댓글, 작성시간과 사이드바, 검색 관련 UI 등의 대부분의 모듈로 삽입되는 html 파일이 들어있는 디렉토리입니다. <BR>
> _javascript : 사용할 모듈을 import 및 불러오는 javascript 파일을 저장하는 디렉토리입니다. 이때 불러온 모듈은 _include 안에서 주로 삽입되어 사용됩니다. <BR>
> _layout : layout 말 그대로, 아케이브, 홈, 포스트, 태그, 카테고리, 페이지 등 블로그에 사용되는 형식을 저장하는 디렉토리입니다. <BR>
> _plugins : RUBY 파일로 포스트 글 변경 시 변경날짜를 저장하는 hook 을 제공하는 plugin 이 들어있는 디렉토리입니다. <BR>
> _posts : 포스트 글 작성하는 디렉토리입니다. <BR>
> _sass : 사용하는 css 파일을 저장하는 디렉토리입니다. 여기서 원하시는 css 를 수정할 수 있습니다. <BR>
> _site : 로컬에서의 화면 UI가 html 파일로 들어있습니다. 로컬에서 파일 수정 후 빌드 시 내용이 변경되지만, _site 내 파일만 변경 시 실제 웹페이지에서 반영되지는 않습니다. <BR>
> _tabs :  기본적인 탭 메뉴 구성 (about, 아케이브, 카테고리, 태그 등)에 대한 랜딩페이지가 들어있는 디렉토리입니다. 따라서 커스터마이징 없이 about 탭에 접속 시 `Add Markdown syntax content to file _tabs/about.md and it will show up on this page.` 라는 문구를 마주하게 됩니다. <BR>
> assets : 웹사이트에 적용할 기본적인 파일이 위치한 디렉토리로, css, 이미지 파일 등이 있습니다. <BR>
> tools : github에 배포하기 위한 코드 파일이 들어있는 디렉토리입니다. <BR>
> _config.yml : 블로그의 기본 환경 설정 파일입니다. <BR>

글을 작성한 후 아래 규칙을 지켜야 합니다.

> 파일명은 yyyy-mm-dd-제목.md 로, 마크다운 기반으로 포스팅해야 합니다. <BR>
> 작성한 파일은 _posts 디렉토리에 넣습니다. <BR>
> 포스트 상단에 다음과 같이 작성해줍니다. <BR>

```
---
title: TITLE <BR>
date: YYYY-MM-DD HH:MM:SS +/-TTTT <BR>
categories: [TOP_CATEGORIE, SUB_CATEGORIE] <BR>
comments: false <BR>
tags: [TAG]     # TAG 이름은 항상 소문자여야 합니다. <BR>
---
```





# Google Analytics 적용
---
Github Page는 방문자 통계 기능을 제공하지 않기 때문에, Google Analytics 를 적용해 통계를 확인하겠습니다. 시작에 앞서, 구글 아이디가 필요합니다. 

[Google Analytics](https://analytics.google.com/analytics/web/provision/#/provision) 에 접속 후, 시작 페이지에서 `측정 시작` 버튼을 클릭합니다.

<img width="1049" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/2e96bb62-d04c-4755-9762-ddbb7f2f1377">

구글에서 안내해주는 다음과 같은 절차를 통해 Google Analytics 를 적용해보겠습니다.

<img width="638" alt="image" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/f87459db-15a8-4bab-8859-f0cbf1bc8d84">


### 1. 속성 만들기
가장 먼저 속성을 만들어주겠습니다. 속성은 웹사이트 단위로 구분 되는 측정하려는 대상 (웹사이트 및/또는 앱)의 데이터 그룹을 나타냅니다. 계정에 속성을 추가하면 해당 속성의 데이터를 수집하며, 이 속성에는 고유의 추적 코드가 생성됩니다. 속성 내에서 보고서를 조회하고 데이터 수집, 기여 분석, 개인 정보 보호 설정, 제품 연결을 관리할 수 있습니다. 

<center>
<img width="500" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/6184a5aa-b01d-4be3-b3a4-52d4403d322d">
</center>
<center>
<img width="450" alt="image" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/5ab552df-d6a1-4e28-bdcf-559add73669f">
</center>
속성의 이름은 간편하게 블로그 링크로 설정 후, 보고 시간대 및 통화는 대한민국으로 설정했습니다. 
속성 이름은 메인 화면에서 다음과 같이 표시됩니다. 속성 이름은 측정하려는 웹사이트를 통칭하는 별명 정도로 생각해도 좋을 것 같습니다.


### 2. 비즈니스 세부정보
이제 비즈니스 세부정보를 입력하겠습니다.
<center>
<img width="500" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/244288ad-73db-4f8f-b4d9-b41d23c576d0">
</center>

뚜렷한 업종 카테고리와 비즈니스를 가지고 있는게 아닌 개인 블로그이기에 업종 카테고리로는 기타, 비즈니스 규모는 작음으로 설정해주었습니다.

### 3. 비즈니스 목표
다음 단계로 비즈니스 목표를 설정해줍니다.
<center>
<img width="500" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/c2db2ff5-a364-4f82-ba5a-3624a1d63170">
</center>
측정하려는 웹사이트의 목표 (리드 생성, 온라인 판매 증대, 브랜드 인지도 향상, 사용자 행동 검토 등) 을 설정해줍니다. 
기준 보고서 보기로 선택 시 특별한 주제 없이 [수명 주기(획득에서 유지까지 고객 여정의 각 단계)에 관한 보고서 모음](https://support.google.com/analytics/answer/12924233?sjid=2530528426777855001-AP)이 보이도록 설정되는 것 같습니다. 다만, 기준 보고서 보기 선택 시 비즈니스 목표는 표시되지 않습니다.

저는 처음 사용하기 때문에 Google Analytics를 잘 이해하기 위해 여러 커스터마이징이 가능할 것 같은 [비즈니스 목표에 관한 보고서](https://support.google.com/analytics/answer/12924488?sjid=2530528426777855001-AP)로 선택해주었습니다. 수명 주기 보고서(기준 보고서) 선택 시 사용자들이 내 웹사이트 또는 앱을 찾아서 사용하는 방식을 이해하는 데 도움이 되는 다양한 보고서가 포함된다고 합니다. 개인의 기호에 따라 원하시는 걸로 선택해주시면 될 것 같습니다.
![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/edbeea68-8d42-4cb8-aff9-f8373f4fd0e7)
선택된 비즈니스 목표 별로 다음과 같이 측정된 보고서를 볼 수 있습니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/a44bb2a2-52d6-427a-81e1-1d17f33c6994)
또한 컬렉션 맞춤설정을 통해 비즈니스 목표에 따른 보고서를 추가,수정,삭제 및 새 주제 (비즈니스 목표)와 주제에 맞는 보고서를 원하는 대로 생성이 가능합니다.

### 4. 데이터 수집

마지막 단계입니다. 드디어 데이터를 수집하기 위한 설정을 진행해봅시다.

<center>
<img width="500" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/04a19c76-cbc7-406f-a415-660cd039ba5a">
</center>
가장 먼저 수집하려는 데이터의 플랫폼을 선택해줍니다. 저는 웹 기반 블로그이기 때문에 웹으로 선택했습니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/b5cc2c56-fdbd-457d-97d6-889677cfd98f)

이제 스트림을 만들어줍시다.
스트림이란 속성 내에 위치하며, 웹사이트 또는 앱에서 analytics로의 데이터 흐름입니다. 예를 들어, 하나의 비즈니스에서 웹사이트와 앱을 함께 운영한다면 스트림을 통해 데이터 소스를 하나의 속성에 담아 분석할 수 있도록 합니다. 데이터 스트림에는 웹(웹사이트), iOS(iOS 앱), Android(Android 앱) 로 세 가지 유형이 있습니다.
    
스트림 이름은 원하시는 걸로 아무거나 작성해주셔도 됩니다.
추후에 수정 가능합니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/d8aa807a-5e3a-4c0c-9719-70058463b45a)

스트림 만들기를 누르면 다음과 같이 측정 ID 를 포함한 스트림 세부정보를 줍니다. 

<center>
<img width="700" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/e33cedd0-f8ec-49e3-a086-6efa4a0b611d">
</center>

이제 해당 스트림의 측정 ID 코드 조각을 제 웹사이트에 추가해주면 됩니다. 만약 사이트가 Jekyll로 구축되지 않은 경우, 사이트에 있는 모든 HTML 파일의 헤드 섹션에 추적 코드 조각을 추가해줘야 합니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/259aa9dd-df3b-4338-b91f-c123abead336)

제가 사용하는 Chirpy 테마의 경우, _includes 디렉토리 안에 미리 다음과 같은 코드와 함께 google-analytics.html 파일이 존재합니다.

<center>
<img width="350" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/6dd3dd16-8aa5-41e7-89bd-9adc2056fae8">
</center>

따라서 _config.yml 파일에 해당 경로로 tracking_id 값만 추가해주면 사용 가능합니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/f033f552-940c-449a-b30e-8e42400466d0)

만약 Chiry 테마와 같이 analytics 관련 파일을 미리 제공해주지 않았을 시에 google analytics 에서 제공한 측정 ID 코드 조각을 _includes 디렉토리 안에 새로운 파일로 붙여넣습니다. 

<center>
<img width="400" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/ddc5f016-9ea4-4669-aa25-f3edb8eb81ca">
</center>

이후 사이트 레이아웃에 분석을 포함시켜주면 됩니다. Jekyll 기반 사이트일 경우 사이트 기본 레이아웃 파일인 _layouts/default.html에 `{​% include {작성한 파일명}.html %​}` 을 추가하주면 됩니다. 이제 사이트의 모든 페이지에 Google Analytics 추적 코드가 포함됩니다. Jekyll 사이트가 아닐 경우 앞서 설명한대로 측정하려는 HTML 파일 각각에 추적 코드에 추가해주어야 합니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/9d18ba51-e976-4fe6-8e83-2a30e798c7f1)

그럼 다음과 같이 Google Analytics 를 통해 웹사이트 측정 및 분석이 가능해집니다!

# Google Adsense 적용
---

블로그에 광고를 개재해 수익을 창출해보고 싶습니다. 
따라서 광고 중개 서비스로 Google Adsense 를 사용하려 합니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/40542dea-0284-4b66-9a5c-62b71e520067)

[Google Adsense](https://www.google.com/adsense) 에 접속해 먼저 광고를 붙이고자 하는 웹사이트를 등록해줍니다.
웹사이트 URL과 맞춤 도움말 이메일 수신 동의 여부, 수취인 국가와 지역을 설정해 에드선스 사용을 시작해보겠습니다.
![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/3b6bb1a9-5e54-437f-83ef-e9369ba8c813)

일련의 과정을 통해 Google Adsense 를 붙일 수 있을 것 같습니다.

### 1. 사용자 정보 입력
<center>
<img width="450" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/158fcea5-ef2b-4e2c-8f9b-0d94846c3bbf">
</center>

먼저 개인 정보를 입력해줍니다. 거주하는 주소와 우편주소, 이름을 입력한 후 다음 단계로 넘어갑니다.

<span style="color:gray">
PS. 미국 여행 갔을 때도 어디 예약하거나 등록만 하려고 하면 주소 입력하는 걸 참 좋아하더군요.. 대충 써도 잘 모르더만 미국 애들은 주소를 좋아하나봐요 (홈리스는 어떡하라고!)
</span>

### 2. 광고 표시 모습 확인

다음 단계로 넘어가면 광고가 어떻게 표시될지를 설정할 수 있습니다.

<div style="display: grid; 	grid-template-columns: 4fr 1fr;">
<span style="display: flex">
<img src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/b653c16e-8b79-4c40-bf9e-8bb89dfc6e48">
</span>
<span style="display: flex">
<img src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/28c8c29e-0824-4984-88fa-4489e5c9ed1c">
</span>
</div>

### 3.에드센스 사이트 연결

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/bdc34d4b-8010-405c-bb69-353ec4105ff6)

구글에서 제공해주는 해당 코드를 _includes 의 head.html 에 넣어줍니다. 
<center>
<img width="500" src="https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/129b6b3b-60f1-4da2-8760-64c47a44513f">
</center>

저는 기존 head.html 에 맞춰 메타 태그로 넣어주겠습니다.

![image](https://github.com/cotes2020/jekyll-theme-chirpy/assets/102217402/516f8052-00de-4cf5-8798-9d522beef6c2)

이제 push 후 결과를 기다리면 됩니다.

다만, Google Adsense 는 사이트 규모 및 가치 등에 따라 반려한다고 합니다.
이번 글이 2번째 글인 저로써는.. 통과 될 일이 없을 것 같기에 점차 글 작성하면서 승인 완료 되면 사이트 적용 후 내용 추가하도록 하겠습니다.

여기까지 긴 글 읽어주셔서 감사합니다 🙇🏻‍♀️ <BR>
앞으로 즐거운 개발 블로그 생활 해보도록 하겠습니다!

