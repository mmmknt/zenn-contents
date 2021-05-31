---
title: "Renovateのログからリポジトリの中で定義しているdependencyの一覧を出してみる"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dependency"]
published: true
---

# Renovateとは

https://github.com/renovatebot/renovate

>Automated dependency updates. Multi-platform and multi-language.

リポジトリのdependencyを自動で更新してくれる仕組み。複数のプラットフォームや言語に対応している。
本来の用途ならRenovateがライブラリの検知、その差分を適用したPRの作成、自動マージといった流れになるが、本記事の目的ではないのでそのあたりは触れません。

# 全体の流れ

* Renovateをself-hostedで、かつdryRunで実行する
  * 手軽に試したいのとログが見たいのでself-hostedで実行します
  * 更新していないライブラリが大量にある場合は急に大量のPRが登録されても困るので、deyRunで実行します
* 実行結果のログからdependencyの一覧を取得する
  * Renovateのログにリポジトリのdependencyを解析した結果が出力されているのでそれを利用します

## Renovateを実行

### 1. GitHubのパーソナルアクセストークンを取得
あとで設定します。
参考：https://docs.github.com/ja/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token

### 2. 適当な作業ディレクトリに設定ファイルとログディレクトリ作成。
設定ファイル：
   ```
   module.exports = {
        token : "<上で取得したアクセストークン>",
        repositories: [
                "対象のリポジトリ（ex. mmmknt/zenn-contents）"
        ],
        dryRun: true,
        logFile: "/var/log/renovatebot.log"
   }
   ```
 
### 3. Renovateを実行
設定ファイルが `/path/to/your/workdir/config.js` 、ログディレクトリが `/path/to/your/workdir/log` の場合:
```
docker run --rm -v "/path/to/your/workdir/config.js:/usr/src/app/config.js" -v "/path/to/your/workdir/log:/var/log" renovate/renovate
```

出力例：
```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
 INFO: Repository started (repository=mmmknt/XXX)
       "renovateVersion": "25.34.0"
 INFO: Dependency extraction complete (repository=mmmknt/XXX)
       "baseBranch": "master",
       "stats": {
         "managers": {
           "dockerfile": {"fileCount": 1, "depCount": 1},
           "gomod": {"fileCount": 1, "depCount": 1},
           "npm": {"fileCount": 1, "depCount": 6}
         },
         "total": {"fileCount": 3, "depCount": 8}
       }
 INFO: DRY-RUN: Would commit files to branch renovate/grpc-proto-loader-0.x (repository=mmmknt/XXX, branch=renovate/grpc-proto-loader-0.x)
 INFO: DRY-RUN: Would commit files to branch renovate/google.golang.org-grpc-1.x (repository=mmmknt/XXX, branch=renovate/google.golang.org-grpc-1.x)
 INFO: DRY-RUN: Would commit files to branch renovate/webpack-5.x (repository=mmmknt/XXX, branch=renovate/webpack-5.x)
 INFO: DRY-RUN: Would commit files to branch renovate/webpack-cli-4.x (repository=mmmknt/XXX, branch=renovate/webpack-cli-4.x)
 INFO: Repository finished (repository=mmmknt/XXX)
       "durationMs": 76544
```

dependency managerごとのライブラリの数や更新があったライブラリは出力されていますが、一覧は見当たらないですね。

### 4. ログからdependencyの一覧を探す
ログの中に以下のような出力があります。
```
{"name":"renovate","hostname":"345b260528d5","pid":14,"level":20,"logContext":"v7-7dBWle","repository":"mmmknt/XXX","config":{"dockerfile":[{"packageFile":"docker/envoy.Dockerfile","deps":[{"depName":"envoyproxy/envoy","currentValue":"latest","replaceString":"envoyproxy/envoy:latest","autoReplaceStringTemplate":"{{depName}}{{#if newValue}}:{{newValue}}{{/if}}{{#if newDigest}}@{{newDigest}}{{/if}}","datasource":"docker","depType":"final","depIndex":0,"updates":[],"warnings":[],"skipReason":"invalid-value"}]}],"gomod":[{"packageFile":"server/go.mod","constraints":{},"deps":[{"managerData":{"lineNumber":2},"depName":"google.golang.org/grpc","depType":"require","currentValue":"v1.22.0","datasource":"go","depIndex":0,"updates":[{"bucket":"non-major","newVersion":"v1.38.0","newValue":"v1.38.0","releaseTimestamp":"2021-05-19T22:23:13.000Z","newMajor":1,"newMinor":38,"updateType":"minor","branchName":"renovate/google.golang.org-grpc-1.x"}],"warnings":[],"sourceUrl":"https://github.com/grpc/grpc-go","currentVersion":"v1.22.0","isSingleVersion":true,"fixedVersion":"v1.22.0"}]}],"npm":[{"packageFile":"web/js/package.json","deps":[{"depType":"devDependencies","depName":"@grpc/proto-loader","currentValue":"^0.5.1","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"0.5.1","depIndex":0,"updates":[{"bucket":"non-major","newVersion":"0.6.2","newValue":"^0.6.0","releaseTimestamp":"2021-05-11T18:58:45.694Z","newMajor":0,"newMinor":6,"updateType":"minor","isRange":true,"branchName":"renovate/grpc-proto-loader-0.x"}],"warnings":[],"sourceUrl":"https://github.com/grpc/grpc-node","homepage":"https://grpc.io/","currentVersion":"0.5.1","isSingleVersion":false,"fixedVersion":"0.5.1"},{"depType":"devDependencies","depName":"google-protobuf","currentValue":"^3.9.0","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"3.9.0","depIndex":1,"updates":[],"warnings":[],"sourceUrl":"https://github.com/protocolbuffers/protobuf","currentVersion":"3.9.0","fixedVersion":"3.9.0"},{"depType":"devDependencies","depName":"grpc","currentValue":"^1.22.2","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"1.22.2","depIndex":2,"updates":[],"warnings":[],"deprecationMessage":"On registry `https://registry.npmjs.org/`, the \"latest\" version of dependency `grpc` has the following deprecation notice:\n\n`This library will not receive further updates other than security fixes. We recommend using @grpc/grpc-js instead.`\n\nMarking the latest version of an npm package as deprecated results in the entire package being considered deprecated, so contact the package author you think this is a mistake.","sourceUrl":"https://github.com/grpc/grpc-node","homepage":"https://grpc.io/","currentVersion":"1.22.2","fixedVersion":"1.22.2"},{"depType":"devDependencies","depName":"grpc-web","currentValue":"^1.0.5","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"1.0.5","depIndex":3,"updates":[],"warnings":[],"sourceUrl":"https://github.com/grpc/grpc-web","homepage":"https://grpc.io/","currentVersion":"1.0.5","fixedVersion":"1.0.5"},{"depType":"devDependencies","depName":"webpack","currentValue":"^4.35.3","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"4.35.3","depIndex":4,"updates":[{"bucket":"major","newVersion":"5.38.1","newValue":"^5.0.0","releaseTimestamp":"2021-05-27T21:12:50.612Z","newMajor":5,"newMinor":38,"updateType":"major","isRange":true,"branchName":"renovate/webpack-5.x"}],"warnings":[],"sourceUrl":"https://github.com/webpack/webpack","currentVersion":"4.35.3","isSingleVersion":false,"fixedVersion":"4.35.3"},{"depType":"devDependencies","depName":"webpack-cli","currentValue":"^3.1.0","datasource":"npm","prettyDepType":"devDependency","lockedVersion":"3.3.5","depIndex":5,"updates":[{"bucket":"major","newVersion":"4.7.0","newValue":"^4.0.0","releaseTimestamp":"2021-05-06T13:11:14.192Z","newMajor":4,"newMinor":7,"updateType":"major","isRange":true,"branchName":"renovate/webpack-cli-4.x"}],"warnings":[],"sourceUrl":"https://github.com/webpack/webpack-cli","currentVersion":"3.3.5","isSingleVersion":false,"fixedVersion":"3.3.5"}],"packageJsonName":"grpc-web-simple-example","packageFileVersion":"0.1.0","packageJsonType":"app","npmLock":"web/js/package-lock.json","managerData":{},"skipInstalls":true,"constraints":{"npm":"<7"},"lockFiles":["web/js/package-lock.json"]}]},"msg":"packageFiles with updates","time":"2021-05-31T15:27:49.025Z","v":0}
```

一部を取り出してフォーマットするとこんな感じ
パッケージマネージャとその管理下にあるdependencyの情報が含まれています。
```
    "gomod": [
      {
        "packageFile": "server/go.mod",
        "constraints": {},
        "deps": [
          {
            "managerData": {
              "lineNumber": 2
            },
            "depName": "google.golang.org/grpc",
            "depType": "require",
            "currentValue": "v1.22.0",
            "datasource": "go",
            "depIndex": 0,
            "updates": [
              {
                "bucket": "non-major",
                "newVersion": "v1.38.0",
                "newValue": "v1.38.0",
                "releaseTimestamp": "2021-05-19T22:23:13.000Z",
                "newMajor": 1,
                "newMinor": 38,
                "updateType": "minor",
                "branchName": "renovate/google.golang.org-grpc-1.x"
              }
            ],
            "warnings": [],
            "sourceUrl": "https://github.com/grpc/grpc-go",
            "currentVersion": "v1.22.0",
            "isSingleVersion": true,
            "fixedVersion": "v1.22.0"
          }
        ]
      }
    ],
```

このログをフォーマットしつつgrepするとこのようにdependencyの一覧が取り出せます。
```
$echo '<ログの本体は上に記載しているので省略>s' | jq . | grep -e '"depName":' -e '"packageFile":'
        "packageFile": "docker/envoy.Dockerfile",
            "depName": "envoyproxy/envoy",
        "packageFile": "server/go.mod",
            "depName": "google.golang.org/grpc",
        "packageFile": "web/js/package.json",
            "depName": "@grpc/proto-loader",
            "depName": "google-protobuf",
            "depName": "grpc",
            "depName": "grpc-web",
            "depName": "webpack",
            "depName": "webpack-cli",
```

jqでごりごりやれば、出力結果をjsonで扱えるようにフォーマットしたり、現在のバージョンやupdate候補のバージョンを出したりできるので色々使えそうな気はしてますが、ここまで。
