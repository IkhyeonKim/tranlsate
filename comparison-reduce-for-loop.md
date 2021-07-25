## 배열에 대해서 동작하는 다른 접근 방식의 비교

**Original post: [Kent C.Dodds' blog - Array reduce vs chaining vs for loop ](https://kentcdodds.com/blog/array-reduce-vs-chaining-vs-for-loop)**

저는 요즘 저의 디지털 추억들을 옮기고 있습니다. 그 중 제가 해야만 하는 일 중 하나는 저의 모든 사진들을 구글 포토에서 다운로드 해야하는 것이였죠.구글의 정렬 방식이 마음에 들지 않아서, 재정렬을 하고 싶다는 생각이 들었고, 간단한 노드 스크립트를 작성했습니다.

스크립트는 이 포스트의 목적과는 다르기 때문에 아주 디테일 하게 들어가지는 않을겁니다. [(하지만 읽어보기를 원하시면 여기 링크가 있습니다)](https://gist.github.com/kentcdodds/59daa81d46ba51b926a6a2f044aa5ad6)
여기 제가 이야기 하고 싶은 부분이 있습니다 (명확하게 하기 위해 살짝 수정했습니다.)

```javascript
const lines = execSync(`find "${searchPath}" -type f`).toString().split("\n");
const commands = lines
  .map((f) => f.trim())
  .filter(Boolean)
  .map((file) => {
    const destFile = getDestFile(file);
    const destFileDir = path.dirname(destFile);
    return `mkdir -p "${destFileDir}" && mv "${file}" "${destFile}"`;
  });
commands.forEach((command) => execSync(command));
```

기본적으로 이 코드는 리눅스의 find 커맨드를 사용하여, 폴더의 모든 파일들의 목록을 찾고, 스크립트의 결과를 분리하고, 공백을 제거하고, 빈 라인을 제거하고, 파일들을 순회하며 이동시키고, 나머지 커맨들을 실행합니다.

이 스크립트를 트위터에 공유했을 때 사람들은 reduce 함수를 사용하는것을 제안했습니다. 아마 사람들은 성능 최적화 문제로 이 같은 제안을 하는것 같습니다, 왜나하면 줄이기 함수를 사용하면 배열 순회 횟수를 줄일 수 있으니까요 (말장난은 아닙니다!)

자 이제는 좀 더 정확하게 이야기 해보겠습니다. 여기 이 배열에는 5만개의 항목들이 있습니다. 일반적인 UI 개발할 때의 수십개의 항목들 보다는 훨씬 많은 경우입니다. 또 제가 생각했던 부분은 단발성의 스크립트를 실행하는 상황에서 성능 향상은 (정말 느리지 않은 경우에는) 제일 나중에 고려합니다. 제 경우에는 꽤 빠르게 동작했습니다.
또 제일 시간을 많이 차지했던 부분은 배열 순회 과정이 아니라 커맨드를 실행하는 부분이었습니다.

또 어떤 사람들은 제게 Node APIs 들이나 오픈 소스 모듈을 사용하는 것을 제안했습니다. 아마도 크로스 플랫폼도
해결하고 성능도 향상되겠죠. 그들이 틀렸다는 것은 아니지만 단발성의 스크립트를 사용하는 상황에서 "준수한 성능"은
크게 고려하지 않았습니다. 이것은 더 불필요한 제약을 걸어 복잡한 해결책을 적용하는 대표적인 예시입니다.

### With reduce

여기 reduce 함수를 사용했을 때의 코드입니다.

```javascript
const commands = lines.reduce((accumulator, line) => {
  let file = line.trim();
  if (file) {
    const destFile = getDestFile(file);
    const destFileDir = path.dirname(destFile);
    accumulator.push(`mkdir -p "${destFileDir}" && mv "${file}" "${destFile}"`);
  }
  return accumulator;
}, []);
```

자 저는 reduce 함수가 만악의 근원이라고 생각하는 사람은 아닙니다. [(흥미로운 예시들을 확인해보세요 - reduce는 만악의 근원이다!)](https://twitter.com/jaffathecake/status/1213077702300852224) 하지만 저는 제가 코드가 읽기 쉬울 때와 복잡할 때를 구분할 줄 안다고 느끼는데, 이 예시에서는 전적으로 체이닝 예시보다 더 복잡하다고 생각합니다.

### With loop

솔직히 array method 들을 사용한 지 너무 오래되어서 loop 사용해서 코드를 다시 작성하는 것은
꽤 시간이 걸릴 것 같네요 잠시만요... 자 이제 다 됐습니다!

```javascript
const commands = [];
for (let index = 0; index < lines.length; index++) {
  const line = lines[index];
  const file = line.trim();
  if (file) {
    const destFile = getDestFile(file);
    const destFileDir = path.dirname(destFile);
    commands.push(`mkdir -p "${destFileDir}" && mv "${file}" "${destFile}"`);
  }
}
```

이것도 그다지 단순해보이지는 않는군요

Edit: 잠시만요 for..of 덕분에 더 단순하게 만들 수 있을 것 같습니다!

```javascript
const commands = [];
for (const line of lines) {
  const file = line.trim();
  if (!file) continue;
  const destFile = getDestFile(file);
  const destFileDir = path.dirname(destFile);
  commands.push(`mkdir -p "${destFileDir}" && mv "${file}" "${destFile}"`);
}
```

솔직히 전통적인 루프 방식과 그다지 크게 나아졌다고 볼 수는 없을 것 같군요, 하지만 꽤 직관적이라고 생각합니다. 사람들은 전통적인 for loops이 유용하게 사용될 때 이 방식을 평가절하 하는 경향이 있는것 같습니다.

### 마치며

저는 앞으로도 for...of 반복문 또는 체이닝을 사용할 것 같습니다. 만약 여러개 배열을 반복해서 돌아야 하는 성능적인 이슈가 있다면 for..of 반복문을 우선적으로 사용할 것입니다.

저는 reduce 함수를 잘 사용하지는 않지만 때때로 사용해보면서 다른 선택지들과 비교해볼 생각입니다. 꽤 주관적으로 들리지만 코딩에서 많은 부분이 주관적이지 않을까요?

다른분들이 어떻게 생각하시는지 알고 싶습니다. 만약 관심이 있으시면 트윗과 리트윗을 해주세요
