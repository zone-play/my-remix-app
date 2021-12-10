## 一、创建 Remix 应用

```base
npx create-remix@latest

yarn dev
```

## 二、创建文件路由

> `app/root.tsx` Layout 组件中

```base
<li>
   <Link to="/posts">Posts</Link>
</li>
```
> 创建 `app/routes/posts/index.tsx`

```base
export default function Posts() {
  return (
    <div>
       <h1>Posts</h1>
    </div>
  )
}
```

## 三、加载数据

Remix 的路由文件相当于后端的模板视图，也是控制器，Remix 建立在 HTTP 和 HTML 的基础之上，因此整个过程无需在浏览器中使用 JavaScript。这也是 Remix 的创新之处，我目前的理解是 Remix 类似 [Nodejs中间层](https://cdmana.com/2021/01/20210116194350856z.html) ，只是 Remix 将 React 整合到了中间层中。这样做的好处是少了一层，提升性能的同时简化了开发成本，让前端更专注于前端，后端更专注于后端。


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/258ygvjp3pur6xl4d2ze.png)

Step1. 项目根目录下（不是app目录） 创建本地文件模拟数据（真实项目往往是从服务器数据库请求数据）

> posts/my-first-post.md

```base
---
title: My First Post
---

# This is my first post

Isn't it great?
```
> posts/90s-mix-cdr.md

```base
---
title: 90s Mixtape
---

# 90s Mixtape

- I wish (Skee-Lo)
- This Is How We Do It (Montell Jordan)
- Everlong (Foo Fighters)
- Ms. Jackson (Outkast)
- Interstate Love Song (Stone Temple Pilots)
- Killing Me Softly With His Song (Fugees, Ms. Lauryn Hill)
- Just a Friend (Biz Markie)
- The Man Who Sold The World (Nirvana)
- Semi-Charmed Life (Third Eye Blind)
- ...Baby One More Time (Britney Spears)
- Better Man (Pearl Jam)
- It's All Coming Back to Me Now (Céline Dion)
- This Kiss (Faith Hill)
- Fly Away (Lenny Kravits)
- Scar Tissue (Red Hot Chili Peppers)
- Santa Monica (Everclear)
- C'mon N' Ride it (Quad City DJ's)
```


Step2. 创建 `app/post.ts` 这个文件是处理`posts`的模块，在这里模拟请求数据

> 首先安装两个模块 [fron-matter](https://hexo.io/zh-cn/docs/front-matter.html) [tiny-invariant](https://github.com/alexreardon/tiny-invariant#readme)

```base
yarn add front-matter //node模块

yarn add tiny-invariant //类型检查
```

```base
import path from "path";
import fs from "fs/promises";
import parseFrontMatter from "front-matter";
import invariant from "tiny-invariant";

export type Post = {
    slug: string;
    title: string;
};

export type PostMarkdownAttributes = {
  title: string;
};

// 相对于服务器输出而不是源！
const postsPath = path.join(__dirname, "..", "posts");

function isValidPostAttributes(
  attributes: any
): attributes is PostMarkdownAttributes {
  return attributes?.title;
}

export async function getPosts() {
  const dir = await fs.readdir(postsPath);
  return Promise.all(
    dir.map(async filename => {
      const file = await fs.readFile(
        path.join(postsPath, filename)
      );
      const { attributes } = parseFrontMatter(
        file.toString()
      );
      invariant(
        isValidPostAttributes(attributes),
        `${filename} 有错误的元数据！`
      );
      return {
        slug: filename.replace(/\.md$/, ""),
        title: attributes.title
      };
    })
  );
}
```

Step3. `app/routes/posts/index.tsx`

```base
import { Link, useLoaderData } from 'remix';
import { getPosts } from '~/post';
import type { Post } from '~/post';

export const loader = () => {
    return getPosts();
};

export default function Posts() {

    const posts = useLoaderData<Post[]>();

    return (
        <div>
            <h1>Posts</h1>
            <ul>
                {posts.map(post => (
                    <li key={post.slug}>
                        <Link to={post.slug}>{post.title}</Link>
                    </li>
                ))}
            </ul>
        </div>
    )
};
```

## 四、动态路由

Step1. 首先安装 [@types/marked](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/marked) 将 markdown 解析为 HTML

```base
yarn add marked

yarn add @types/marked
```

Step2. 为 `app/post.ts` 模块添加一个函数 `getPost`

```base
import path from "path";
import fs from "fs/promises";
import parseFrontMatter from "front-matter";
import invariant from "tiny-invariant";
import { marked } from 'marked';

export type Post = {
  slug: string;
  title: string;
};

export type PostMarkdownAttributes = {
  title: string;
};

// 相对于服务器输出而不是源！
const postsPath = path.join(__dirname, "..", "posts");

function isValidPostAttributes(
  attributes: any
): attributes is PostMarkdownAttributes {
  return attributes?.title;
};

export async function getPosts() {
  const dir = await fs.readdir(postsPath);
  return Promise.all(
    dir.map(async filename => {
      const file = await fs.readFile(
        path.join(postsPath, filename)
      );
      const { attributes } = parseFrontMatter(
        file.toString()
      );
      invariant(
        isValidPostAttributes(attributes),
        `${filename} 有错误的元数据！`
      );
      return {
        slug: filename.replace(/\.md$/, ""),
        title: attributes.title
      };
    })
  );
};

export async function getPost(slug: string) {
  const filepath = path.join(postsPath, slug + ".md");
  const file = await fs.readFile(filepath);
  const { attributes, body } = parseFrontMatter(
    file.toString()
  );
  invariant(
    isValidPostAttributes(attributes),
    `Post ${filepath} is missing attributes`
  );
  const html = marked(body);
  return { slug, html, title: attributes.title };
}
```

Step3. 创建动态路由文件

> app/routes/posts/$slug.tsx

```base
import { useLoaderData } from 'remix';
import type { LoaderFunction } from 'remix';
import { getPost } from "~/post";
import invariant from "tiny-invariant";

export const loader: LoaderFunction = async ({ params }) => {
    invariant(params.slug, "expected params.slug");
    return getPost(params.slug);
};

export default function PostSlug() {

    const post = useLoaderData();

    return (
        <div dangerouslySetInnerHTML={{ __html: post.html }} />
    );

};
```

## 五、子路由（嵌套路由）

Step1. 创建一个 admin 路由文件

> `app/routes/admin.tsx` 注意使用：Outlet

```base
import { Outlet, Link, useLoaderData } from "remix";
import { getPosts } from "~/post";
import type { Post } from "~/post";
import adminStyles from "~/styles/admin.css";

export const links = () => {
  return [{ rel: "stylesheet", href: adminStyles }];
};

export const loader = () => {
  return getPosts();
};

export default function Admin() {
  const posts = useLoaderData<Post[]>();
  return (
    <div className="admin">
      <nav>
        <h1>Admin</h1>
        <ul>
          {posts.map(post => (
            <li key={post.slug}>
              <Link to={`/posts/${post.slug}`}>
                {post.title}
              </Link>
            </li>
          ))}
        </ul>
      </nav>
      <main>
          <Outlet/>
      </main>
    </div>
  );
};
```
> `app/styles/admin.css`

```base
.admin {
    display: flex;
  }
  
  .admin > nav {
    padding-right: 2rem;
  }
  
  .admin > main {
    flex: 1;
    border-left: solid 1px #ccc;
    padding-left: 2rem;
  }
  
  em {
    color: red;
  }
```

Step2. 为 admin.tsx 创建子路由文件夹 

> `app/routes/admin/index.tsx`

```base
import { Link } from "remix";

export default function AdminIndex() {
  return (
    <p>
      <Link to="new">Create a New Post</Link>
    </p>
  );
}
```

Step3. 在 `app/post.ts` 处理新建Post的逻辑

```base
import path from "path";
import fs from "fs/promises";
import parseFrontMatter from "front-matter";
import invariant from "tiny-invariant";
import { marked } from 'marked';

export type Post = {
  slug: string;
  title: string;
};

export type PostMarkdownAttributes = {
  title: string;
};

type NewPost = {
  title: string;
  slug: string;
  markdown: string;
};

// 相对于服务器输出而不是源！
const postsPath = path.join(__dirname, "..", "posts");

function isValidPostAttributes(
  attributes: any
): attributes is PostMarkdownAttributes {
  return attributes?.title;
};


//Post列表
export async function getPosts() {
  const dir = await fs.readdir(postsPath);
  return Promise.all(
    dir.map(async filename => {
      const file = await fs.readFile(
        path.join(postsPath, filename)
      );
      const { attributes } = parseFrontMatter(
        file.toString()
      );
      invariant(
        isValidPostAttributes(attributes),
        `${filename} 有错误的元数据！`
      );
      return {
        slug: filename.replace(/\.md$/, ""),
        title: attributes.title
      };
    })
  );
};

// Post详情
export async function getPost(slug: string) {
  const filepath = path.join(postsPath, slug + ".md");
  const file = await fs.readFile(filepath);
  const { attributes, body } = parseFrontMatter(
    file.toString()
  );
  invariant(
    isValidPostAttributes(attributes),
    `Post ${filepath} is missing attributes`
  );
  const html = marked(body);
  return { slug, html, title: attributes.title };
};

// 新建Post
export async function createPost(post: NewPost) {
  const md = `---\ntitle: ${post.title}\n---\n\n${post.markdown}`;
  await fs.writeFile(
    path.join(postsPath, post.slug + ".md"),
    md
  );
  return getPost(post.slug);
};
```

Step4. 创建 `app/routes/admin/new.tsx`

```base
import { useTransition, useActionData, Form, redirect } from "remix";
import type { ActionFunction } from "remix";
import { createPost } from "~/post";
import invariant from "tiny-invariant";

type PostError = {
    title?: boolean;
    slug?: boolean;
    markdown?: boolean;
};

export const action: ActionFunction = async ({
    request
}) => {
    await new Promise(res => setTimeout(res, 1000));

    const formData = await request.formData();

    const title = formData.get("title");
    const slug = formData.get("slug");
    const markdown = formData.get("markdown");

    const errors: PostError = {};
    if (!title) errors.title = true;
    if (!slug) errors.slug = true;
    if (!markdown) errors.markdown = true;

    if (Object.keys(errors).length) {
        return errors;
    }

    invariant(typeof title === "string");
    invariant(typeof slug === "string");
    invariant(typeof markdown === "string");
    await createPost({ title, slug, markdown });

    return redirect("/admin");
};

export default function NewPost() {

    const errors = useActionData();
    const transition = useTransition();

    return (
        <Form method="post">
            <p>
                <label>
                    Post Title:{" "}
                    {errors?.title && <em>Title is required</em>}
                    <input type="text" name="title" />
                </label>
            </p>
            <p>
                <label>
                    Post Slug:{" "}
                    {errors?.slug && <em>Slug is required</em>}
                    <input type="text" name="slug" />
                </label>
            </p>
            <p>
                <label htmlFor="markdown">Markdown:</label>{" "}
                {errors?.markdown && <em>Markdown is required</em>}
                <br />
                <textarea id="markdown" rows={20} name="markdown" />
            </p>
            <p>
                <button type="submit">
                    {transition.submission
                        ? "Creating..."
                        : "Create Post"}
                </button>
            </p>
        </Form>
    );
}
```
