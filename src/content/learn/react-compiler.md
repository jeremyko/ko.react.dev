---
title: React 컴파일러
---

<Intro>
이 페이지는 React 컴파일러에 대한 소개와 이를 성공적으로 시도하는 방법을 제공합니다.
</Intro>

<YouWillLearn>

* 컴파일러 시작하기
* 컴파일러 및 ESLint 플러그인 설치
* 문제 해결

</YouWillLearn>

<Note>

React 컴파일러는 커뮤니티로부터 초기 피드백을 받기 위해 RC<sup>Release Candidate</sup> 버전으로 오픈소스화한 새로운 컴파일러입니다. 이제 모든 분께 이 컴파일러를 사용해 보고 피드백을 제공할 것을 권장합니다.

최신 RC 릴리스는 `@rc` 태그로 찾을 수 있고, 일일 실험적 릴리스는 `@experimental`로 찾을 수 있습니다.
</Note>

React 컴파일러는 커뮤니티로부터 초기 피드백을 받기 위해 오픈소스화한 새로운 컴파일러입니다. React 컴파일러는 빌드 타임 전용 도구로 React 앱을 자동으로 최적화합니다. 순수 자바스크립트로 동작하며, [React의 규칙](/reference/rules)을 이해하므로 코드를 다시 작성할 필요가 없습니다.

`eslint-plugin-react-hooks`에는 코드 에디터에서 컴파일러의 분석 결과를 즉시 보여주는 [ESLint 규칙](#installing-eslint-plugin-react-compiler)도 포함되어 있습니다. **지금 당장 모든 분들들께 이 린터 사용을 강력히 권장합니다.** 린터는 컴파일러의 설치가 필요 없으므로 컴파일러를 사용할 준비가 되지 않았더라도 사용할 수 있습니다.

컴파일러는 현재 `rc` 버전으로 출시되었으며, React 17 이상의 앱과 라이브러리에서 사용해 볼 수 있습니다. `rc` 버전을 설치하려면 다음 명령어를 실행하세요.

<TerminalBlock>
{`npm install -D babel-plugin-react-compiler@rc eslint-plugin-react-hooks@^6.0.0-rc.1`}
</TerminalBlock>

또는, Yarn을 사용한다면 아래 명령어를 실행해주세요.

<TerminalBlock>
{`yarn add -D babel-plugin-react-compiler@rc eslint-plugin-react-hooks@^6.0.0-rc.1`}
</TerminalBlock>

React 19를 아직 사용하지 않는다면, 추가 지침을 위해 [아래 섹션](#using-react-compiler-with-react-17-or-18)을 확인하세요.

### 컴파일러는 무엇을 하나요? {/*what-does-the-compiler-do*/}

React 컴파일러는 애플리케이션을 최적화하기 위해 코드를 자동으로 메모이제이션<sup>Memoization</sup>합니다. 이미 `useMemo`, `useCallback`, `React.memo`와 같은 API를 통한 메모이제이션에 익숙할 것입니다. 이러한 API를 사용하면 React에게 입력이 변경되지 않았다면 특정 부분을 다시 계산할 필요가 없다고 알릴 수 있어 업데이트 시 작업량을 줄일 수 있습니다. 이 방법은 강력하지만 메모이제이션을 적용하는 것을 잊거나 잘못 적용할 수도 있습니다. 이 경우 React가 _의미 있는_ 변경 사항이 없는 UI 일부를 확인해야 하므로 효율적이지 않을 수 있습니다.

컴파일러는 자바스크립트와 React의 규칙에 대한 지식을 활용하여 자동으로 컴포넌트와 Hook 내부의 값 또는 값 그룹을 메모이제이션 합니다. 규칙 위반을 감지할 경우 해당 컴포넌트 또는 Hook을 건너뛰고 다른 코드를 안전하게 컴파일합니다.

<Note>
React 컴파일러는 React의 규칙 위반을 정적으로 감지할 수 있으며, 영향을 받는 컴포넌트나 Hook만 안전하게 최적화에서 제외할 수 있습니다. 컴파일러가 코드베이스의 100%를 최적화할 필요는 없습니다.
</Note>

이미 코드베이스에 메모이제이션이 잘 되어 있다면, 컴파일러를 통해 주요 성능 향상을 기대하기 어려울 수 있습니다. 그러나 실제로 성능 문제를 일으키는 올바른 의존성<sup>dependencies</sup>을 메모이제이션 하는 것은 직접 처리하기 까다로울 수 있습니다.

<DeepDive>
#### React Compiler는 어떤 것을 메모이제이션 하나요? {/*what-kind-of-memoization-does-react-compiler-add*/}

React 컴파일러의 초기 릴리즈는 주로 **업데이트 성능 개선**(기존 컴포넌트의 리렌더링)에 초점을 맞추었으므로 다음 두 가지 사용 사례에 중점을 두고 있습니다.

1. **컴포넌트의 연쇄적인 리렌더링 건너뛰기**
    * `<Parent />`를 리렌더링하면 `<Parent />`만이 변경되었음에도 불구하고 그 컴포넌트 트리 내의 여러 컴포넌트가 리렌더링되는 경우.
2. **React 외부에서의 비용이 많이 드는 계산 건너뛰기**
    * 데이터가 필요한 컴포넌트나 Hook 내에서 `expensivelyProcessAReallyLargeArrayOfObjects()`를 호출하는 경우.

#### 리렌더링 최적화 {/*optimizing-re-renders*/}

React는 Props, State, Context와 같은 현재 State에 대한 함수로 UI를 표현할 수 있도록 해줍니다. `useMemo()`, `useCallback()`, 또는 `React.memo()`로 수동 메모이제이션을 적용하지 않은 경우에 현재 구현에서 컴포넌트의 State가 변경되면, React는 해당 컴포넌트와 <em>하위 모든 자식 컴포넌트</em>를 리렌더링합니다. 예를 들어 다음 예시에서는 `<FriendList>`의 State가 변경될 때마다 `<MessageButton>`이 리렌더링됩니다.

```javascript
function FriendList({ friends }) {
  const onlineCount = useFriendOnlineCount();
  if (friends.length === 0) {
    return <NoFriends />;
  }
  return (
    <div>
      <span>{onlineCount} online</span>
      {friends.map((friend) => (
        <FriendListCard key={friend.id} friend={friend} />
      ))}
      <MessageButton />
    </div>
  );
}
```
[_React 컴파일러 플레이그라운드에서 이 예시를 확인하세요_](https://playground.react.dev/#N4Igzg9grgTgxgUxALhAMygOzgFwJYSYAEAYjHgpgCYAyeYOAFMEWuZVWEQL4CURwADrEicQgyKEANnkwIAwtEw4iAXiJQwCMhWoB5TDLmKsTXgG5hRInjRFGbXZwB0UygHMcACzWr1ABn4hEWsYBBxYYgAeADkIHQ4uAHoAPksRbisiMIiYYkYs6yiqPAA3FMLrIiiwAAcAQ0wU4GlZBSUcbklDNqikusaKkKrgR0TnAFt62sYHdmp+VRT7SqrqhOo6Bnl6mCoiAGsEAE9VUfmqZzwqLrHqM7ubolTVol5eTOGigFkEMDB6u4EAAhKA4HCEZ5DNZ9ErlLIWYTcEDcIA).

React 컴파일러는 상태 변경 시 앱에서 관련된 부분만 리렌더링되도록 수동 메모이제이션과 동등한 기능을 자동으로 적용합니다. 이를 "세분화된 반응성<sup>Fine-Grained Reactivity</sup>"이라고도 부릅니다. 위 예시에서 React 컴파일러는 `friends`가 변경되더라도 `<FriendListCard />`의 반환 값이 재사용될 수 있음을 결정하고, JSX를 재생성하지 않고 `<MessageButton>`의 리렌더링도 피할 수 있습니다.

#### 비용이 많이 드는 계산 메모이제이션 {/*expensive-calculations-also-get-memoized*/}

컴파일러는 렌더링 도중 비용이 많이 드는 계산에 대해 자동으로 메모이제이션을 적용할 수도 있습니다.

```js
// 컴포넌트나 Hook이 아니기 때문에 React 컴파일러에 의해 **메모이제이션 되지 않습니다.**
function expensivelyProcessAReallyLargeArrayOfObjects() { /* ... */ }

// 컴포넌트이기 때문에 React 컴파일러에 의해 메모이제이션 됩니다.
function TableContainer({ items }) {
  // 이 함수 호출은 메모이제이션 될 것입니다.
  const data = expensivelyProcessAReallyLargeArrayOfObjects(items);
  // ...
}
```
[_React 컴파일러 플레이그라운드에서 이 예시를 확인하세요_](https://playground.react.dev/#N4Igzg9grgTgxgUxALhAejQAgFTYHIQAuumAtgqRAJYBeCAJpgEYCemASggIZyGYDCEUgAcqAGwQwANJjBUAdokyEAFlTCZ1meUUxdMcIcIjyE8vhBiYVECAGsAOvIBmURYSonMCAB7CzcgBuCGIsAAowEIhgYACCnFxioQAyXDAA5gixMDBcLADyzvlMAFYIvGAAFACUmMCYaNiYAHStOFgAvk5OGJgAshTUdIysHNy8AkbikrIKSqpaWvqGIiZmhE6u7p7ymAAqXEwSguZcCpKV9VSEFBodtcBOmAYmYHz0XIT6ALzefgFUYKhCJRBAxeLcJIsVIZLI5PKFYplCqVa63aoAbm6u0wMAQhFguwAPPRAQA+YAfL4dIloUmBMlODogDpAA).

그러나 `expensivelyProcessAReallyLargeArrayOfObjects`가 실제로 비용이 많이 드는 함수라면 다음과 같은 이유로 React 외부에서 해당 함수의 별도 메모이제이션을 고려해야 할 수도 있습니다.

- React 컴파일러는 React 컴포넌트와 Hook만 메모이제이션 하며, 모든 함수를 메모이제이션 하지 않습니다.
- React 컴파일러의 메모이제이션은 여러 컴포넌트나 Hook 사이에서 공유되지 않습니다.

따라서 `expensivelyProcessAReallyLargeArrayOfObjects`가 여러 다른 컴포넌트에서 사용되고 있다면 동일한 아이템이 전달되더라도 비용이 많이 드는 계산이 반복적으로 실행될 수 있습니다. 코드를 더 복잡하게 만들기 전에 먼저 [프로파일링](/reference/react/useMemo#how-to-tell-if-a-calculation-is-expensive)을 통해 해당 계산이 실제로 비용이 많이 드는지 확인하는 것이 좋습니다.
</DeepDive>

### 컴파일러를 시도해 봐야 하나요? {/*should-i-try-out-the-compiler*/}

컴파일러는 현재 Release Candidate(RC) 단계에 있으며, 이미 프로덕션 환경에서 광범위하게 테스트되었습니다. Meta와 같은 회사에서는 이미 프로덕션 환경에 도입하여 사용하고 있지만, 앱에 컴파일러를 점진적으로 도입할지는 코드베이스의 건강 상태와 [React의 규칙](/reference/rules)을 얼마나 잘 준수했는지에 따라 달라질 것입니다.

**지금 당장 컴파일러를 사용하기에 급급할 필요는 없습니다. 안정적인 릴리즈에 도달할 때까지 기다려도 괜찮습니다.** 하지만 앱에서의 작은 실험을 통해 컴파일러를 시도해 보고 [피드백을 제공](#reporting-issues)하여 컴파일러 개선에 도움을 줄 수 있습니다.

## 시작하기 {/*getting-started*/}

현재 문서 외에도 [React 컴파일러 워킹 그룹](https://github.com/reactwg/react-compiler)을 확인하여 컴파일러에 대한 추가 정보와 논의를 참조하는 것을 권장합니다.

### `eslint-plugin-react-hooks` 설치 {/*installing-eslint-plugin-react-compiler*/}

React 컴파일러는 ESLint 플러그인도 제공합니다. `eslint-plugin-react-hooks@^6.0.0-rc.1`을 설치해서 시도해보세요.

<TerminalBlock>
{`npm install -D eslint-plugin-react-hooks@^6.0.0-rc.1`}
</TerminalBlock>

자세한 정보를 위해 [에디터 설정하기](/learn/editor-setup#linting) 가이드를 확인하세요.

ESLint 플러그인은 에디터에서 React의 규칙 위반 사항을 표시합니다. 이 경우 컴파일러가 해당 컴포넌트나 Hook의 최적화를 건너뛰었음을 의미합니다. 이것은 완전히 정상적인 동작이며, 컴파일러는 이를 복구하고 코드베이스의 다른 컴포넌트를 계속해서 최적화할 수 있습니다.

<Note>
**모든 eslint 위반 사항을 즉시 수정할 필요는 없습니다.** 자신의 속도에 맞춰 해결하면서 최적화되는 컴포넌트와 Hook의 수를 늘릴 수 있지만, 컴파일러를 사용하기 전에 모든 것을 수정해야 할 필요는 없습니다.
</Note>

### 코드베이스에 컴파일러 적용하기 {/*using-the-compiler-effectively*/}

#### 기존 프로젝트 {/*existing-projects*/}

컴파일러는 [React의 규칙](/reference/rules)을 따르는 함수 컴포넌트와 Hook을 컴파일하는 것을 목표로 설계되었습니다. 또한 이러한 규칙을 위반하는 코드도 해당 컴포넌트나 Hook을 건너뛰는 방식으로 처리할 수 있습니다. 그러나 자바스크립트의 유연한 특성으로 인해 컴파일러가 가능한 모든 위반 사항을 잡아내지는 못하며, 가끔 거짓 음성<sup>False Negatives</sup>으로 컴파일할 수 있습니다. 즉, 컴파일러는 React의 규칙을 위반하는 컴포넌트나 Hook을 실수로 컴파일할 수 있어 정의되지 않은 동작으로 이어질 수 있습니다.

따라서 기존 프로젝트에서 컴파일러를 성공적으로 도입하려면, 먼저 제품 코드의 작은 디렉토리에서 실행해 보는 것이 좋습니다. 이를 위해 컴파일러를 특정 디렉토리 집합<sup>Set</sup>에서만 실행하도록 구성할 수 있습니다.

```js {3}
const ReactCompilerConfig = {
  sources: (filename) => {
    return filename.indexOf('src/path/to/dir') !== -1;
  },
};
```

컴파일러를 도입하는 데 더 자신감을 가지게 되면, 다른 디렉터리에 대한 커버리지를 확대하고 점진적으로 전체 앱에 적용할 수 있습니다.

#### 새로운 프로젝트 {/*new-projects*/}

새 프로젝트를 시작할 경우, 기본 동작으로 전체 코드베이스에서 컴파일러를 활성화할 수 있습니다.

### React 17 또는 18에서 컴파일러 사용 {/*using-react-compiler-with-react-17-or-18*/}

React 컴파일러는 React 19 RC에서 최적의 성능을 발휘합니다. 업그레이드가 불가능하다면 `react-compiler-runtime` 패키지를 추가 설치해 컴파일된 코드를 19 이전 버전에서도 실행할 수 있습니다. 단, 최소 지원 버전은 17입니다.

<TerminalBlock>
{`npm install react-compiler-runtime@rc`}
</TerminalBlock>

컴파일러 설정에 올바른 `target`을 추가해야 하며, `target`은 대상으로 하는 React의 메이저 버전입니다.

```js {3}
// babel.config.js
const ReactCompilerConfig = {
  target: '18' // '17' | '18' | '19'
};

module.exports = function () {
  return {
    plugins: [
      ['babel-plugin-react-compiler', ReactCompilerConfig],
    ],
  };
};
```

### 라이브러리에서 컴파일러 사용 {/*using-the-compiler-on-libraries*/}

React 컴파일러는 라이브러리 컴파일에도 사용할 수 있습니다. React 컴파일러는 코드 변환 이전의 원본 소스 코드에서 실행되어야 하기 때문에, 애플리케이션의 빌드 파이프라인으로는 사용하는 라이브러리를 컴파일할 수 없습니다. 따라서, 라이브러리 유지보수자가 컴파일러로 라이브러리를 독립적으로 컴파일하고 테스트한 후, 컴파일된 코드를 npm에 배포하는 것을 권장합니다.

코드가 사전 컴파일되기 때문에, 라이브러리 사용자들은 라이브러리에 적용된 자동 메모이제이션의 이점을 얻기 위해 컴파일러를 활성화할 필요가 없습니다. 라이브러리가 아직 React 19 버전이 아닌 앱을 대상으로 한다면, 최소 [`target`을 설정하고 `react-compiler-runtime`을 직접 의존성으로 추가하세요](#using-react-compiler-with-react-17-or-18). 런타임 패키지는 애플리케이션 버전에 따라 올바른 API 구현을 사용하며, 필요한 경우 누락된 API를 폴리필합니다.

라이브러리 코드는 종종 더 복잡한 패턴과 탈출구 사용이 필요할 수 있습니다. 이런 이유로, 라이브러리에 컴파일러를 적용할 때 발생할 수 있는 문제를 발견하기 위해 충분한 테스트를 갖추는 것을 권장합니다. 문제를 발견하면, 해당 컴포넌트나 훅을 [`'use no memo'` 지시어](#something-is-not-working-after-compilation)를 통해 언제든 제외할 수 있습니다.

앱과 비슷하게, 라이브러리에서 이점을 보기 위해 모든 컴포넌트나 훅을 100% 컴파일할 필요는 없습니다. 좋은 시작점은 라이브러리에서 성능에 민감한 부분을 파악하고 해당 부분이 [React의 규칙](/reference/rules)을 위반하지 않는지 확인하는 것입니다. 이는 `eslint-plugin-react-compiler`를 통해 파악할 수 있습니다.

## 사용법 {/*installation*/}

### Babel {/*usage-with-babel*/}

<TerminalBlock>
{`npm install babel-plugin-react-compiler@rc`}
</TerminalBlock>

컴파일러에는 빌드 파이프라인에서 사용할 수 있는 Babel 플러그인이 포함되어 있습니다.

설치 후 Babel 설정<sup>Config</sup> 파일에 추가하세요. 파이프라인에서 컴파일러가 **먼저** 실행되는 것이 가장 중요합니다.

```js {7}
// babel.config.js
const ReactCompilerConfig = { /* ... */ };

module.exports = function () {
  return {
    plugins: [
      ['babel-plugin-react-compiler', ReactCompilerConfig], // 가장 먼저 실행하세요!
      // ...
    ],
  };
};
```

`babel-plugin-react-compiler`는 다른 Babel 플러그인보다 먼저 실행되어야 합니다. 이는 컴파일러가 사운드 분석<sup>Sound Analysis</sup>을 위해 입력 소스 정보를 필요로 하기 때문입니다.

### Vite {/*usage-with-vite*/}

Vite를 사용하고 있다면, `vite-plugin-react`에 플러그인을 추가할 수 있습니다.

```js {10}
// vite.config.js
const ReactCompilerConfig = { /* ... */ };

export default defineConfig(() => {
  return {
    plugins: [
      react({
        babel: {
          plugins: [
            ["babel-plugin-react-compiler", ReactCompilerConfig],
          ],
        },
      }),
    ],
    // ...
  };
});
```

### Next.js {/*usage-with-nextjs*/}

자세한 정보를 위해 [Next.js 문서](https://nextjs.org/docs/app/api-reference/next-config-js/reactCompiler)를 확인하세요.

### Remix {/*usage-with-remix*/}
`vite-plugin-babel`을 설치하고 컴파일러의 Babel 플러그인을 추가하세요.

<TerminalBlock>
{`npm install vite-plugin-babel`}
</TerminalBlock>

```js {2,14}
// vite.config.js
import babel from "vite-plugin-babel";

const ReactCompilerConfig = { /* ... */ };

export default defineConfig({
  plugins: [
    remix({ /* ... */}),
    babel({
      filter: /\.[jt]sx?$/,
      babelConfig: {
        presets: ["@babel/preset-typescript"], // TypeScript를 사용하는 경우.
        plugins: [
          ["babel-plugin-react-compiler", ReactCompilerConfig],
        ],
      },
    }),
  ],
});
```

### Webpack {/*usage-with-webpack*/}

커뮤니티 webpack 로더가 [이제 여기에서 사용 가능합니다](https://github.com/SukkaW/react-compiler-webpack).

### Expo {/*usage-with-expo*/}

Expo 앱에서 React 컴파일러를 활성화하고 사용하려면 [Expo 문서](https://docs.expo.dev/guides/react-compiler/)를 확인하세요.

### Metro (React Native) {/*usage-with-react-native-metro*/}

React Native는 Metro를 통해 Babel을 사용하므로 설치 지침은 [Babel 사용법](#usage-with-babel) 섹션을 참조하세요.

### Rspack {/*usage-with-rspack*/}

Rspack 앱에서 React 컴파일러를 활용하거나 사용하기 위해서는 [Rspack's docs](https://rspack.dev/guide/tech/react#react-compiler)를 참고해주세요.

### Rsbuild {/*usage-with-rsbuild*/}

Rsbuild 앱에서 React 컴파일러를 활용하거나 사용하기 위해서는 [Rsbuild's docs](https://rsbuild.dev/guide/framework/react#react-compiler)를 참고해주세요.

## Troubleshooting {/*troubleshooting*/}

문제를 보고하려면 먼저 [React 컴파일러 플레이그라운드](https://playground.react.dev/)에서 최소한의 재현 사례를 만들어 버그 보고서에 포함하세요. [facebook/react](https://github.com/facebook/react/issues) 저장소에서 이슈를 열 수 있습니다.

React 컴파일러 워킹 그룹에서도 회원으로 지원하여 피드백을 제공할 수 있습니다. [가입에 대한 자세한 내용은 README](https://github.com/reactwg/react-compiler)에서 확인하세요.

### 컴파일러는 무엇을 가정하나요? {/*what-does-the-compiler-assume*/}

React 컴파일러는 다음과 같이 가정합니다.

1. 올바르고 의미 있는 자바스크립트 코드로 작성되었습니다.
2. nullable/optional 값과 속성에 접근하기 전에 그 값이 정의되어 있는지 테스트합니다. 예를 들어, TypeScript를 사용하는 경우 [`strictNullChecks`](https://www.typescriptlang.org/ko/tsconfig/#strictNullChecks)을 활성화하여 수행합니다. 즉, `if (object.nullableProperty) { object.nullableProperty.foo }`와 같이 처리하거나, 옵셔널 체이닝<sup>Optional Chaning</sup>을 사용하여 `object.nullableProperty?.foo`와 같이 처리합니다.
3. [React의 규칙](/reference/rules)을 따릅니다.

React 컴파일러는 React의 많은 규칙을 정적으로 검증할 수 있으며, 에러가 감지되면 안전하게 컴파일을 건너뜁니다. 에러를 확인하려면 [`eslint-plugin-react-compiler`](https://www.npmjs.com/package/eslint-plugin-react-compiler)의 설치를 권장합니다.

### 컴포넌트가 최적화되었는지 어떻게 알 수 있을까요? {/*how-do-i-know-my-components-have-been-optimized*/}

[React DevTools](/learn/react-developer-tools) (v5.0+)와 [React Native DevTools](https://reactnative.dev/docs/react-native-devtools)는 React 컴파일러를 내장 지원하며, 컴파일러에 의해 최적화된 컴포넌트 옆에 "Memo ✨" 배지를 표시합니다.

### 컴파일 후 작동하지 않는 문제 {/*something-is-not-working-after-compilation*/}

`eslint-plugin-react-compiler`을 설치한 경우, 컴파일러는 에디터에서 React 규칙 위반 사항을 표시합니다. 이 경우 컴파일러가 해당 컴포넌트나 Hook의 최적화를 건너뛰었음을 의미합니다. 이것은 완전히 정상적인 동작이며, 컴파일러는 이를 복구하고 코드베이스의 다른 컴포넌트를 계속해서 최적화할 수 있습니다. **모든 ESLint 위반 사항을 즉시 수정할 필요는 없습니다.** 자신의 속도에 맞춰 해결하면서 최적화되는 컴포넌트와 Hooks의 수를 점진적으로 늘릴 수 있습니다.

그러나 자바스크립트의 유연하고 동적인 특성 때문에 모든 경우를 철저하게 감지하는 것은 불가능합니다. 이러면 버그나 무한 루프와 같은 정의되지 않은 동작이 발생할 수 있습니다.

컴파일 후 앱이 제대로 작동하지 않고 ESLint 에러도 보이지 않는다면, 컴파일러가 코드를 잘못 컴파일한 것일 수 있습니다. 이를 확인하려면 관련된 컴포넌트나 Hook을 [`"use no memo"` 지시어](#opt-out-of-the-compiler-for-a-component)를 통해 강력하게 제외해 문제를 해결하려고 시도해 보세요.

```js {2}
function SuspiciousComponent() {
  "use no memo"; // 컴포넌트가 React 컴파일러에 의해 컴파일되지 않도록 제외합니다.
  // ...
}
```

<Note>
#### `"use no memo"` {/*use-no-memo*/}

`"use no memo"`는 React 컴파일러에 의해 컴파일되지 않도록 컴포넌트와 Hooks를 선택적으로 제외할 수 있는 _임시_ 탈출구입니다. 이 지시어는 [`"use client"`](/reference/rsc/use-client)와 같이 장기적으로 사용하지 않을 임시방편입니다.

이 지시어는 필요한 경우가 아니면 사용을 권장하지 않습니다. 한 번 컴포넌트나 Hook을 제외하면 해당 지시어가 제거될 때까지 영구적으로 컴파일에서 제외합니다. 이는 코드를 수정해도 컴파일러가 해당 부분을 여전히 건너뛸 것을 의미합니다.
</Note>

문제를 해결했을 때 지시어를 제거하면 문제가 다시 발생하는지 확인하세요. 그런 다음 [React 컴파일러 플레이그라운드](https://playground.react.dev)를 활용하여 문제를 최소한의 재현 가능한 예시로 단순화해 보거나, 오픈 소스 코드라면 전체 소스 코드를 붙여 넣어 버그 보고서를 공유해주세요. 이를 통해 문제를 파악하고 해결하는 데 도움을 드릴 수 있습니다.

### 이외의 문제 {/*other-issues*/}

자세한 내용은 https://github.com/reactwg/react-compiler/discussions/7 을 참조해 주세요.
