---
title: React Router를 사용하면 최상위 ErrorBoundary까지 에러가 전달되지 않는다고요?
description: React Router를 사용하면 최상위 ErrorBoundary까지 에러가 전달되지 않습니다. 이 문제를 해결하는 방법에 대해 알아봅니다.
date: 2024-12-12 23:30:00 +0900
categories: [React, React Router]
tags: [error boundary, react-router, 에러처리, 오류처리]
toc: true
toc_sticky: true
toc_label: 목차
pin: true
math: true
mermaid: true
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---

# 문제

개발을 진행하던 중, 컴포넌트에서 에러가 발생했는데, `Unexpected Application Error` 가 발생하는 것을 발견했습니다. 이상한 점은 해당 프로젝트의 최상단에는 `ErrorBoundary` 가 감싸져 있는 상태였는데, 해당 `ErrorBoundary`에 설정된 Fallback UI가 노출되지 않았다는 점이었어요. 이상함을 느껴서 디버깅을 해보니, 최상위에 감싸진 `ErrorBoundary` 까지 에러가 도달하지 않는 다는 것을 알 수 있었습니다. 

확인 해보니, 최상위 `ErrorBoundary` 가 있고, react-router의 `createBrowserRouter` 와 `RouterProvider` 를 사용하는 경우, 라우터 하위 컴포넌트에서 발생하는 에러가 최상위 `ErrorBoundary` 까지 전달되지 않는 문제가 있었습니다. 

react-router를 사용하는 방식은 2가지가 있는데, `createBrowserRouter` 와 `RouterProvider`를 사용하는 방식과 `<BrowserRouter>` 를 사용하는 방식이 있습니다. 두 가지 방법 모두 확인해봤더니, `<BrowserRouter>`를 사용하는 방식에서는 최상위 `ErrorBoundary` 까지 에러가 전달되는 것을 확인할 수 있었습니다. 

```tsx
// createBrowserRouter를 사용하는 방식 - 이슈 발생 🔴
const router = createBrowserRouter([
  {
    path: "/",
    element: <MainPage />,
  },
  {
    path: "/sub",
    element: <SubPage />,
  },
]);

export default function App() {
  return (
    <ErrorBoundary>
      <RouterProvider router={router} />
    </ErrorBoundary>
  );
}

// <BrowserRouter>를 사용하는 방식 - 이슈 발생하지 않음 🟢
export default function App() {
  return (
    <ErrorBoundary>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<MainPage />} />
          <Route path="/sub" element={<SubPage />} />
        </Routes>
      </BrowserRouter>
    </ErrorBoundary>
  );
}
```

간단한 테스트 코드를 통해 상황을 재현해 볼 수 있습니다. 

심플한 React 앱을 세팅하고, react-router를 설치해준 후, 테스트 코드를 작성해보겠습니다. 

*(React 앱 세팅과 react-router 설치에 대한 설명은 생략하겠습니다)*

```tsx
// ErrorBoundary.tsx

interface ErrorBoundaryProps {
  children?: ReactNode;
}

interface ErrorBoundaryState {
  error: Error | null;
  hasError: boolean;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = {
    error: null,
    hasError: false,
  };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.log(error);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
```

```tsx
function MainPage() {
  return (
    <div>
      <h1>MainPage</h1>
      <Link to="/sub">Link to Sub Page</Link>
    </div>
  );
}

function SubPage() {
  // SubPage 접속 시, 1초 뒤 에러 발생
  const [isError, setIsError] = useState(false);
  useEffect(() => {
    setTimeout(() => {
      setIsError(true);
    }, 1000);
  }, []);
  if (isError) throw new Error("error!!");
  return <h1>SubPage</h1>;
}

export default function App() {
  return (
    <ErrorBoundary>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<MainPage />} />
          <Route path="/sub" element={<SubPage />} />
        </Routes>
      </BrowserRouter>
    </ErrorBoundary>
  );
}
```

우선, `<BrowserRouter>` 를 사용하는 방식으로 테스트를 해보겠습니다. SubPage로 이동 시, 1초 뒤 에러가 발생하게 세팅해두었습니다. 

`/sub` 에 접속하면, 아래와 같이 최상위 `ErrorBoundary` 에 세팅한 Something went wrong 이라는 문구가 노출됩니다. 

![image.png](/assets/img/react-router/something-went-wrong.png)

그럼 이번에는 `createBrowserRouter` 와 `RouterProvider` 를 사용해보겠습니다.

```tsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <MainPage />,
  },
  {
    path: "/sub",
    element: <SubPage />,
  },
]);

export default function App() {
  return (
    <ErrorBoundary>
      <RouterProvider router={router} />
    </ErrorBoundary>
  );
}

```

`/sub` 페이지를 접속해보면, 아래와 같은 화면이 노출되는 것을 확인할 수 있습니다. 

![image.png](/assets/img/react-router/unexpected-error.png)

이 화면은 dev server에서만 노출되는 화면이고, 실 서비스에서는 사용자가 빈 화면을 보게 될겁니다. 

# 원인

현상을 확인했으니, 원인을 찾아보기 위해 [react-router 레포지토리](https://github.com/remix-run/react-router)를 방문했습니다. 아마 이런 현상을 겪은 사람이 많을 것이라고 생각했죠. 아니나 다를까, 이미 관련된 Discussion 이 있었습니다. 

https://github.com/remix-run/react-router/discussions/10166

`createBrowserRouter` 와 `RouterProvider` 를 사용하는 경우, 라우터 상위의 `ErrorBoundary` 로 에러처리를 하지 못하고, react-router 단에서 에러를 먹어(?) 버리는 문제에 대해서 논의가 이뤄졌었습니다. 다만 많은 사람이 문제를 제기하고 문제에 동의했는데도 불구하고, 메인테이너 측에서는 현재의 설계가 적절하다는 입장인 것 같았습니다. 🥲

![image.png](/assets/img/react-router/discussion.png)

결론적으로 원인은 react-router의 자체 `Error Boundary`였습니다. `createBrowserRouter` 와 `RouterProvider` 를 사용하면 react-router의 자체 `ErrorBoundary` 로 감싸게 되어있고, 에러를 throw 하지 않기 때문에, 상위의 `ErrorBoundary` 까지 에러가 전달되지 않습니다.


[react-router 의 RenderErrorBoundary](https://github.com/remix-run/react-router/blob/5d96537148d768b304be3bea7237a12351127807/packages/react-router/lib/hooks.tsx#L668)

```tsx
export class RenderErrorBoundary extends React.Component<
  RenderErrorBoundaryProps,
  RenderErrorBoundaryState
> {
  constructor(props: RenderErrorBoundaryProps) {
    super(props);
    this.state = {
      location: props.location,
      revalidation: props.revalidation,
      error: props.error,
    };
  }

  static getDerivedStateFromError(error: any) {
    return { error: error };
  }

  static getDerivedStateFromProps(
    props: RenderErrorBoundaryProps,
    state: RenderErrorBoundaryState
  ) {
    // When we get into an error state, the user will likely click "back" to the
    // previous page that didn't have an error. Because this wraps the entire
    // application, that will have no effect--the error page continues to display.
    // This gives us a mechanism to recover from the error when the location changes.
    //
    // Whether we're in an error state or not, we update the location in state
    // so that when we are in an error state, it gets reset when a new location
    // comes in and the user recovers from the error.
    if (
      state.location !== props.location ||
      (state.revalidation !== "idle" && props.revalidation === "idle")
    ) {
      return {
        error: props.error,
        location: props.location,
        revalidation: props.revalidation,
      };
    }

    // If we're not changing locations, preserve the location but still surface
    // any new errors that may come through. We retain the existing error, we do
    // this because the error provided from the app state may be cleared without
    // the location changing.
    return {
      error: props.error !== undefined ? props.error : state.error,
      location: state.location,
      revalidation: props.revalidation || state.revalidation,
    };
  }

  componentDidCatch(error: any, errorInfo: any) {
    console.error(
      "React Router caught the following error during render",
      error,
      errorInfo
    );
  }

  render() {
    return this.state.error !== undefined ? (
      <RouteContext.Provider value={this.props.routeContext}>
        <RouteErrorContext.Provider
          value={this.state.error}
          children={this.props.component}
        />
      </RouteContext.Provider>
    ) : (
      this.props.children
    );
  }
}

```

# 해결

이 문제를 해결하기 위해 react-router Route의 `errorElement` props를 사용할 수 있습니다. 

errorElement props는 JSX.Element 타입으로 react-router의 자체 ErrorBoundary에서는 errorElement를 Fallback Element로 사용하게 됩니다. 

```tsx
<Route
  path="/"
  element={<Home />}
  errorElement={<div>Somthing went wrong..</div>}
/>
```

errorElement는 각 Route마다 설정해줄 수도 있고, errorElement가 설정되어 있지 않으면, 상위 Route의 errorElement를 찾게 됩니다. errorElement를 통해 uncaught error로 처리되지 않고 적절한 에러 화면이 노출되도록 설정해줄 수 있습니다. 

하지만 저는 errorElement로 에러 화면을 보여주고 싶은 것이 아니라, 최상위에 제가 감싸놓은 ErrorBoundary까지 에러가 전달되어 처리되길 원했습니다. 최상위 ErrorBoundary에 모니터링 도구로 에러 로그를 전송하는 등의 처리가 되어있었기 때문입니다.

그래서 저는 아래와 같이 errorElement에서 상위로 에러를 throw 해주는 방법을 사용했습니다. 

```tsx
// RouteErrorElement.tsx
import { isRouteErrorResponse, useRouteError } from 'react-router';

export const RouterErrorElement = () => {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    // 라우트 에러인 경우 에러 화면을 보여줍니다.
    return <RouteErrorFallback />;
  }
  // 상위 에러 바운더리로 throw!
  throw error;
};

// App.tsx
...
<Route
  path="/"
  element={<Home />}
  errorElement={<RouteErrorElemet />}
/>
...
```

`useRouterError()` hooks를 이용하여 error 객체를 받아올 수 있습니다. 그리고 `isRouteErrorResponse` 를 이용하여 라우트 에러인지 판단하고 라우트 에러인 경우에는 에러 화면을 보여줍니다. 그리고 에러를 상위 ErrorBoundary로 throw 합니다.

이렇게 하면 의도한 대로 최상위의 ErrorBoundary에서 에러를 처리할 수 있습니다.

# 마치며

위에 언급한 디스커션에서 많은 사람들이 문제를 제기한대로, 저도 React Router의 자체 ErrorBoundary가 예측불가능하게 구현되어 있어, 사용자에게 의도치 않은 문제를 일으켰다고 생각합니다. `errorElement` 라는 props를 제공하고 있기는 하지만, errorElement 를 따로 설정하지 않는다면 상위로 에러가 throw 되도록 하는 것이 적절한 설계가 아닌가 생각되네요. 

그리고 이 문제가 일관되지 않은 것도 문제라고 생각됩니다. `createBrowserRouter` 를 사용하는 것이 아니라, `<BrowserRouter>` 를 사용하는 경우에는 문제가 발생하지 않았거든요. 이런 설계가 적절한 설계라고 생각한다면, 사용 방법에 따라서도 일관되게 적용 되었어야 하지 않나 싶습니다. 

react router 라는 라이브러리가 정말 많이 사용되고 있는 라이브러리인데.. 조금은 실망을 하게 된 것 같습니다. 

그리고 이번 이슈를 겪으며 오랜만에 React Router Docs를 살펴보게 되었는데, React Router가 V7이 되면서 프레임워크 형태로 제공하고 (거의 Remix화 되는 것 같은..) 있어, Docs도 거의 프레임워크 형태에 대한 내용이 주를 이루고 있었습니다. 라이브러리 형태로 사용하는 것에 대한 문서도 있긴 있었지만 굉장히 기본적인 내용에만 국한되어 조금만 작성되어 있었습니다. React Router를 프레임워크로 사용하는 사람보단 라이브러리로 라우팅만 사용하는 사람이 많을텐데, 좀 아쉬운 부분이지 않나 싶네요. Next.js에 대항하기 위함일까요.

이제는 React로 웹앱을 만드는건 Next.js 나 Remix 같은 메타 프레임워크로 진행하는 것이 일반화가 되어가고 있는 것 같습니다. React 공식 홈페이지에서도 메타 프레임워크 사용을 권장하고 있으니깐요. 

작은 이슈였고 짧은 해결 과정이었지만, 많은 걸 느끼게 되었던 것 같습니다.