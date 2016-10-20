---
layout: post
title: Window에서 Electron Packaging부터 Installer까지 (with Grunt)
tags: Electron build Grunt
categories: Javascript
---
Electron을 앱을 배포하기 위해서는 앱을 Packaging후 깔끔하게 installer 파일로 bulid하여 배포하는게 좋을텐데요. 여기서는 기본적인 Packaging부터 Installer Build, Grunt를 이용해 빌드 자동화 까지 해보도록 하겠습니다.

## Electron Build Process
Electron Build 과정은 크게 Packaging, Asar 압축, Installer Build 세 단계 정도로 나눠볼 수 있습니다. 좀더 살을 붙이자면 아래와 같습니다.
* Packaging : 프로젝트를 **electron-packager** 모듈을 이용해 기본 패킹하는 과정
* Asar 압축 : Packaging후 resources의 App폴더를 Asar파일로 압축
* Installer Bulid : Packaging후 나온 App을 Installer 하나의 파일로 만드는 과정

## Electron Build 준비물
우선 전체 과정에 필요한 모듈을 설치합니다. 각 모듈은 아래와 같습니다.
* electron-packager : electron 패키징 모듈
* electron-installer-squirrel-windows : 윈도우용 일렉트론 인스톨러 빌드 모듈
* Asar : asar 압축 지원 모듈
{% highlight js %}
  npm i --save-dev electron-packager electron-installer-squirrel-windows asar
{% endhighlight %}

## Electron Packaging
[Electron Packaging](https://github.com/electron-userland/electron-packager)모듈을 통해 프로젝트를 패키징 합니다.
{% highlight js %}
electron-packager . TestApp --platform win32 --arch x64 --out dist/
{% endhighlight %}
위의 과정을 거치면 현재 폴더에 dist/TestApp-win32-x64 폴더하위로 프로젝트가 패키징 된 것을 볼 수 있습니다. 이상태로 TestApp-win32-x64폴더내의 TestApp.exe파일을 실행하면 앱이 실행됩니다.

## Asar 압축
[Asar](https://github.com/electron/asar)은 Electron에서 지원하는 압축 포맷입니다. dist/TestApp-win32-x64/resources폴더를 보시면 app폴더를 보실 수 있으며 여기엔 프로젝트의 코드가 그대로 존재합니다. Asar압축은 이 app폴더를 app.asar파일로 만들어주는 과정입니다.
Asar 압축 명령은 다음과 같습니다.
{% highlight js %}
asar p ./dist/OpenClib-win32-x64/resources/app.asar ./dist/OpenClib-win32-x64/resources/app
{% endhighlight %}
압축이후 기존의 app폴더는 지워준후 실행파일을 실행해보면 실행이 잘되는것을 보실수 있습니다.

위의 두과정은 한데 묶을수 있습니다. electron-packager를 실행시 --asar옵션을 추가해주면 됩니다.
{% highlight js %}
electron-packager . TestApp --asar --platform win32 --arch x64 --out dist/
{% endhighlight %}

## Installer build
[electron-installer-squirrel-windows](https://www.npmjs.com/package/electron-installer-squirrel-windows)모듈을 통해 패키지를 인스톨러로 빌드합니다. electron을 빌드해주는 다른 모듈들이 있지만 약간씩 문제가 있어 이 모듈을 사용합니다. 이 모듈은 쉘커맨드를 사용할 수도 있지만 여기서는 installer.js라는 인스톨 환경설정파일을 만들어서 적용해 보겠습니다.
{% highlight js %}
// installer.js
var createInstaller = require('electron-installer-squirrel-windows');

createInstaller({
  name : 'TestApp',
  path: './dist/TestApp-win32-x64',
  out: './dist/installer',
  authors: 'creamyCode',
  exe: 'TestApp.exe',
  appDirectory: './dist/TestApp-win32-x64',
  overwrite: true,
  setup_icon: 'favicon.ico'
}, function done (e) {
  console.log('Build success !!');
});
{% endhighlight %}

실행은 간단히
{% highlight js %}
node Installer.js
{% endhighlight %}

# Grunt 를 통한 빌드 자동화
## Grunt ?
[Grunt](http://gruntjs.com/)는 태스크 기반 **Javascript build tool** 입니다. Grunt는 기본적으로 사용자가 작성한 Gruntfile의 Task기반으로 작동되며 이 태스크는 minification, compilation, unit testing, linting등등이 있습니다.

## Grunt 설치
우선 grunt-cli를 설치하겠습니다. CLI는 Command Line Interface의 줄임말이며 콘솔에서 grunt를 사용할 수 있게 해주는 녀석입니다.
{% highlight js %}
  npm i -g grunt-cli
{% endhighlight %}

다음은 프로젝트에 grunt 및 grunt로 쉘 커맨드를 실행할 수 있는 grunt-exec모듈를 설치합니다.
{% highlight js %}
  npm i --save-dev grunt grunt-exec
{% endhighlight %}

## Grunt 작업설정
Grunt 작업 설정은 **Gruntfile.js** 를 통해 할 수 있으며 다음은 일렉트론 패키징 및 인스톨러 빌드의 기본적인 부분으로 작성된 형태입니다.
{% highlight js %}
module.exports = function(grunt) {

// 플러그인 명세
  [
    'grunt-exec'
  ].forEach(function(task){
    grunt.loadNpmTasks(task);   // grunt 플러그인 등록
  });

  // Project configuration.
  grunt.initConfig({
    pkg: grunt.file.readJSON('package.json'),
      // grunt-exec 플러그인 작업 명세
      'exec': {
        // 일렉트론 패키징 과정
        'x64-packager': {
          command: "electron-packager . TestApp --overwrite --asar --platform win32 --arch x64 --out dist/ --icon favicon.ico --ignore=.gitignore --ignore=Gruntfile.js --ignore=webpack.config.js --ignore=installer.js",
          stdout: true,
          stderr: true
        },
        // 일렉트론 인스톨러 빌드 과정
        'x64-installer': {
          command: "node ./installer.js x64",
          stdout: true,
          stderr: true
        }
      }

  });

  // 작업 등록, 배열내의 인덱스 차례로 플러그인 관련작업이 동작
  grunt.registerTask('default', ['exec']);

};
{% endhighlight %}

## Grunt 실행
실행은 매우 간단합니다. 다음과 같이 실행해주시면 됩니다.
{% highlight js %}
  grunt
{% endhighlight %}
