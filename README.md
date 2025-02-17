# codemod-replace-react-fc-typescript

A codemod using [jscodeshift](https://github.com/facebook/jscodeshift) to remove `React.FC` and `React.SFC` from your codebase

![gif animation showing how this codemod removes React.FC](https://github.com/gndelia/codemod-replace-react-fc-typescript/blob/main/example.gif?raw=true)

(It's recommended to run your favorite formatting tool after the codemod :wink: )

> 💡 Running the codemod [with Intuita](#using-intuita) will automatically format your code with prettier.

## :man_teacher: Motivation

IF you use React and Typescript, you might have come across this [GitHub PR in Create React App's repo](https://github.com/facebook/create-react-app/pull/8177) about removing `React.FC` from their base template of a Typescript project.

The three main points that made me buy this was the fact that:

- There's an implicit definition of `children` - all your components will have `children` typed!
- They don't support generics
- It does not correctly work with `defaultProps`

as well as other downsides (check out the PR description for that)

Motivated by that PR, and a lot of blog posts who also shared similar conclusions (and this [ADR](https://backstage.io/docs/architecture-decisions/adrs-adr006) from Spotify's team in which they recorded the decision of removing `React.FC` from they codebase too), I wrote this little codemod that drops `React.FC`, `React.FunctionComponent` and `React.SFC` (which was also deprecated) and replaces the Props as the type of the unique argument in the component definition.

Let's see it with code

```tsx
// before codemod runs
type Props2 = { id: number };
export const MyComponent2: React.FC<Props2> = (props) => {
  return <span>{props.id}</span>
}

// after codemod runs
type Props2 = { id: number };
export const MyComponent2 = (props: Props2) => {
  return <span>{props.id}</span>
}

```

It also works if the Props are defined inline

```tsx
// before codemod runs
export const MyComponent4: React.FC<{ inlineProp: number, disabled?: boolean }> = (props) => <span>foo</span>

// after codemod runs
export const MyComponent4 = (
  props: {
    inlineProp: number,
    disabled?: boolean
  }
) => <span>foo</span>
```

It works with generics too!

```tsx
// before codemod runs
type GenericsProps<T extends any> = { config: T }
export const MyComponentWithGenerics: React.FC<GenericsProps<string>> = (props) => <span>{props.config}</span>
export const MyComponentWithGenerics2: React.FC<GenericsProps<{ text: string }>> = ({ config: { text }}) => <span>{text}</span>

// after codemod runs
type GenericsProps<T extends any> = { config: T }
export const MyComponentWithGenerics = (props: GenericsProps<string>) => <span>{props.config}</span>
export const MyComponentWithGenerics2 = (
  {
    config: { text }
  }: GenericsProps<{ text: string }>
) => <span>{text}</span>
```

and with props defined with intersection

```tsx
// before codemod runs
const WithIntersection: React.FC<Props1 & Props2> = ({ id, ...restProps }) => <span>{id}</span>

// after codemod runs
const WithIntersection = ( { id, ...restProps }: Props1 & Props2 ) => <span>{id}</span>
```

and with component modules defined using intersection

```tsx
// before codemod runs
import React from 'react';
import { OtherComponent } from "./other-component";

interface Props { text: string }
const WithComponentIntersection: React.FC<Props> & {
  OtherComponent: typeof OtherComponent;
} = (props) => {
  return <span>{props.text}</span>
}
WithComponentIntersection.OtherComponent = OtherComponent;

// after codemod runs
import React from 'react';
import { OtherComponent } from "./other-component";

interface Props { text: string }
const WithComponentIntersection = (props: Props) => {
  return <span>{props.text}</span>
}
WithComponentIntersection.OtherComponent = OtherComponent;
```

Even with no Props!

```tsx
// before codemod runs
const NoPropsComponent: React.FC = () => <span>foo</span>

// after codemod runs
const NoPropsComponent = () => <span>foo</span>
```

You don't have to stick with arrow functions only; all the previous scenarios work with regular named functions as well

```tsx
// before codemod runs
import React from 'react'

interface Props { text: string }
const HelloWorld: React.SFC<Props> = function HelloWorld(props) {
  return <div>Hi {props.someValue}</div>
}

// after codemod runs
import React from 'react'

interface Props { text: string }
const HelloWorld = function HelloWorld(props: Props) {
  return <div>Hi {props.someValue}</div>
}
```

It also works when you use a function that accepts a component definition

```ts
// before codemod runs
import React from 'react';
import { observer } from "mobx-react-lite";

type Props = { id: number };
const functionAcceptsComponent: React.FC<Props> = observer((props) => {
  return <span>{props.id}</span>
})

// after codemod runs
import React from 'react';
import { observer } from "mobx-react-lite";

type Props = { id: number };
export const functionAcceptsComponent = observer((props: Props) => {
  return <span>{props.id}</span>
})
```

This codemod also works when using `FC`, `FunctionComponent` and `SFC` as a named export

```tsx
// before codemod runs
import React, { FC } from 'react'

const NamedExportComponent: FC<Props> = (props) => <span>foo</span>

// after codemod runs
import React, { FC } from 'react'

const NamedExportComponent = (props: Props) => <span>foo</span>
```

## :toolbox: How to use

### Using Intuita

![gif animation showing how to use Intuita to run this codemod](/intuita-example.gif?raw=true)

<div align="center">

  To run the codemod with the Intuita VS Code extension, [install Intuita VS Code extension](https://marketplace.visualstudio.com/items?itemName=Intuita.intuita-vscode-extension) and run the codemod:</p>
  
<div><a href="https://tinyurl.com/replace-react-fc-typescript" target="_blank"><img src="https://raw.githubusercontent.com/intuita-inc/intuita-docs/master/static/img/misc/1-click-run-codemod-with-intuita-badge.png" alt="Intuita Run Codemod" height="45px"/></a></div>
</div>

> 💡 To learn more about running codemods using Intuita, check the usage guide [here](https://docs.intuita.io/docs/vs-code-extension/advanced-usage).

### Using `jscodeshift`

Run the following command

```
npx jscodeshift -- -t https://raw.githubusercontent.com/gndelia/codemod-replace-react-fc-typescript/main/dist/index.js --extensions=tsx --verbose=2 <FOLDER-YOU-WANT-TO-TRANSFORM>
```

There are other options you can read in the jscodeshift's Readme.

`jscodeshift` only accepts local transform files, or remote self-contained files. That's why I compiled the transform file into one distributable file using [@vercel/ncc](https://github.com/vercel/ncc). If you don't want to run this remote file (because you might not trust, although you can read the source - it is totally safe), you can download this repo and run

```
npx jscodeshift -- -t Path/To/Repo/transform.ts --extensions=tsx --verbose=2 <FOLDER-YOU-WANT-TO-TRANSFORM>
```

## :notebook: Notes

- The codemod focuses on replacing the nodes but does not do styling. If you run the codemod using `jscodeshift` CLI, you might want to run Prettier or your favorite formatting tool after the code has been modified. However, if you run the codemod using intuita, you can safely skip this part as Intuita automatically formats the output with prettier.

### Example:

```tsx
import React from 'react'

interface Props { id: number, text: string }
const Component: React.FC<Props> = (props) => (
  <div>
    <span>{props.id}</span>
  </div>
)
```

after running the codemod, you might lose the parenthesis

```tsx
import React from 'react'

interface Props { id: number, text: string }
const Component = (props: Props) => <div>
  <span>{props.id}</span>
</div>
```

this is because those parenthesis are not strictly required for the code to work. You can fix this by running `Prettier` (or whatever tool you're using to format your code) easily, as the code is still valid



- If your component was using the implicit definition of `children` provided by `React.FC`, you will have to add the explicit definition or the code won't compile. For example, the following code

```tsx
import React from 'react'

type Props = { title: string }
const Component: React.FC<Props> = ({ title, children }) => <div title={title}>{children}</div>
```

will be transformed into this after running the codemod
```tsx
import React from 'react'

type Props = { title: string }
const Component = ({ title, children }: Props) => <div title={title}>{children}</div>
```

However, it won't compile because `children` is not part of your `Props` definition anymore. You can solve this by manually adding the type of `children` again.

The value that `React.FC` provides (that accepts anything you would accept in js as children) is `{ children?: ReactNode }`. I'm intentionally not automatically adding it because  you can restrict it to what you only want to accept (for instance, just a string, a number, only one component, and so on), and you know better than I do what you need.


## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/gndelia"><img src="https://avatars1.githubusercontent.com/u/352474?v=4?s=100" width="100px;" alt="Gonzalo D'Elia"/><br /><sub><b>Gonzalo D'Elia</b></sub></a><br /><a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=gndelia" title="Code">💻</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=gndelia" title="Tests">⚠️</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=gndelia" title="Documentation">📖</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/pulls?q=is%3Apr+reviewed-by%3Agndelia" title="Reviewed Pull Requests">👀</a> <a href="#infra-gndelia" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://liby.github.io"><img src="https://avatars.githubusercontent.com/u/38807139?v=4?s=100" width="100px;" alt="Bryan Lee"/><br /><sub><b>Bryan Lee</b></sub></a><br /><a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=liby" title="Tests">⚠️</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/issues?q=author%3Aliby" title="Bug reports">🐛</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=liby" title="Documentation">📖</a> <a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=liby" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://jlarky.now.sh/"><img src="https://avatars.githubusercontent.com/u/7026?v=4?s=100" width="100px;" alt="Yaroslav Lapin"/><br /><sub><b>Yaroslav Lapin</b></sub></a><br /><a href="https://github.com/gndelia/codemod-replace-react-fc-typescript/commits?author=JLarky" title="Documentation">📖</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
