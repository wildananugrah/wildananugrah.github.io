# frontend

1. create folder frontend
2. npx create-next-app@latest .
3. npm i --save @react-oauth/google antd universal-cookie
4. create .env

```sh
NEXT_PUBLIC_HOST_BE=http://localhost:5020
NEXT_PUBLIC_GOOGLE_CLIENT_ID=419461063850-7a5l66s7uhmp8hev97mc7j31r8ai7720.apps.googleusercontent.com
```

5. modify styles
6. create src/components
7. delete src/styles/global.css
8. create src/components/register folder
9. create src/components/register/helpers folder
10. create src/components/register/helpers/register.helper.js

```js
export async function registerByGmail(accessToken) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_HOST_BE}/register/gmail`,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    }
  );
  const responseJson = await response.json();
  if (!response.ok) throw new Error(responseJson.message);
  return responseJson;
}
```

11. create src/components/register/Register.jsx

```jsx
import { Col, Row } from "antd";
import { GoogleOAuthProvider } from "@react-oauth/google";
import RegisterForm from "./RegisterForm";

const Register = () => {
  return (
    <GoogleOAuthProvider clientId={process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID}>
      <Row>
        <Col span={6} offset={9}>
          <div
            style={{
              marginTop: 200,
              boxShadow: "5px 5px 10px #CCC",
              padding: 10,
              borderRadius: 10,
            }}
          >
            <RegisterForm />
          </div>
        </Col>
      </Row>
    </GoogleOAuthProvider>
  );
};

export default Register;
```

12. create src/components/register/RegisterForm.jsx

```jsx
import Link from "next/link";
import { Form, Space, Button, message } from "antd";
import { useGoogleLogin } from "@react-oauth/google";
import { registerByGmail } from "./helpers/register.helper";
import { useRouter } from "next/router";
import Cookies from "universal-cookie";

const RegisterForm = () => {
  const router = useRouter();
  const onGoogleLogin = useGoogleLogin({
    onSuccess: async (codeResponse) => {
      try {
        const responseJson = await registerByGmail(codeResponse.access_token);
        message.success(responseJson.message);
        const cookies = new Cookies();
        cookies.set("token", JSON.stringify(responseJson));
        router.push("/admin/dashboard");
      } catch (e) {
        console.error(e.message);
        message.error(e.message);
      }
    },
  });

  return (
    <Form
      initialValues={{
        remember: true,
      }}
      autoComplete="off"
    >
      <div
        style={{
          display: "flex",
          justifyContent: "center",
          alignItems: "center",
          height: 100,
        }}
      >
        <h1>Logo</h1>
      </div>

      <Form.Item>
        <div
          style={{
            display: "flex",
            justifyContent: "center",
            alignItems: "center",
            height: 100,
          }}
        >
          <Space direction="horizontal">
            <Button
              onClick={(e) => {
                e.preventDefault();
                onGoogleLogin();
              }}
            >
              Register with Google
            </Button>
            <Link href="/login">Already have an account ?</Link>
          </Space>
        </div>
      </Form.Item>
    </Form>
  );
};
export default RegisterForm;
```

13. create src/components/login
14. create src/components/login/helpers
15. create src/components/login/helpers/login.helpers.js

```js
export async function loginByGmail(accessToken) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_HOST_BE}/login/gmail`,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    }
  );
  const responseJson = await response.json();
  if (!response.ok) throw new Error(responseJson.message);
  return responseJson;
}
```

16. create src/components/login/LoginForm.jsx

```jsx
import Link from "next/link";
import { Form, Space, Button, message } from "antd";
import { loginByGmail, onSuccessCallback } from "./helpers/login.helper.js";
import { useGoogleLogin } from "@react-oauth/google";
import Cookies from "universal-cookie";
import { useRouter } from "next/router.js";

const LoginForm = () => {
  const router = useRouter();
  const onGoogleLogin = useGoogleLogin({
    onSuccess: async (codeResponse) => {
      try {
        const responseJson = await loginByGmail(codeResponse.access_token);
        message.success(responseJson.message);
        const cookies = new Cookies();
        cookies.set("token", JSON.stringify(responseJson));
        router.push("/admin/dashboard");
      } catch (e) {
        console.error(e.message);
        message.error(e.message);
      }
    },
  });

  return (
    <Form
      initialValues={{
        remember: true,
      }}
      autoComplete="off"
    >
      <div
        style={{
          display: "flex",
          justifyContent: "center",
          alignItems: "center",
          height: 100,
        }}
      >
        <h1>Logo</h1>
      </div>

      <Form.Item>
        <div
          style={{
            display: "flex",
            justifyContent: "center",
            alignItems: "center",
            height: 100,
          }}
        >
          <Space direction="horizontal">
            <Button
              onClick={(e) => {
                e.preventDefault();
                onGoogleLogin();
              }}
            >
              Sign in with Google
            </Button>
            <Link href="/register">Don't have an account ?</Link>
          </Space>
        </div>
      </Form.Item>
    </Form>
  );
};
export default LoginForm;
```

17. create src/components/login/Login.jsx

```jsx
import { Col, Row } from "antd";
import LoginForm from "./LoginForm";
import { GoogleOAuthProvider } from "@react-oauth/google";

const Login = () => {
  return (
    <GoogleOAuthProvider clientId={process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID}>
      <Row>
        <Col span={6} offset={9}>
          <div
            style={{
              marginTop: 200,
              boxShadow: "5px 5px 10px #CCC",
              padding: 10,
              borderRadius: 10,
            }}
          >
            <LoginForm />
          </div>
        </Col>
      </Row>
    </GoogleOAuthProvider>
  );
};

export default Login;
```

18. create src/pages/index.js

```js
import { useRouter } from "next/router";
import { useEffect } from "react";

export default () => {
  const router = useRouter();
  useEffect(() => {
    router.push("/login");
  }, []);
};
```

19. create src/pages/register.js

```js
import Register from "@/components/register/Register";
export default Register
```

20. create src/pages/login.js

```js
import Login from "@/components/login/Login";
export default Login
```

21. create src/pages/logout.js

```js
import { useRouter } from "next/router";
import { useEffect } from "react";
import Cookies from "universal-cookie";

export default () => {
  const router = useRouter();
  useEffect(() => {
    new Cookies().remove("token");
    router.push("/login");
  }, []);
};
```
