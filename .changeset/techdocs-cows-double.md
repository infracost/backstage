---
'@backstage/plugin-techdocs': minor
---

The `mkdocs` is now a replaceable option on your `TechDocsReaderPage` so you can get inspired and create other reader implementations that better fit with your Tech Docs API implementation:

**app.tsx**

```tsx
import { TechDocsReaderPage } from '@backstage/plugin-techdocs';
import { techDocsPage } from '@backstage/plugin-techdocs-mkdocs'; 👈
 ...
<Route path="/:kind/:namespace/:name" element={<TechDocsReaderPage />}>
  {techDocsPage}
</Route>
...
```

If you don't pass a `techDocsPage` we use the `mkdocs` implementation by default, i.e. _these changes should not cause breaking changes_:

**app.tsx**

```tsx
import { TechDocsReaderPage } from '@backstage/plugin-techdocs';
...
<Route path="/:kind/:namespace/:name" element={<TechDocsReaderPage />} />
...
```

The `@backstage/plugin-techdocs` package is also exporting a new component called `TechDocsShadowDom` which can be used by future reader developers if they want to render documentation content using the [Shadow Root API](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot).

```tsx
import {
  useTechDocsReader,
  TechDocsShadowDom,
} from '@backstage/plugin-techdocs';

export const CustomTechDocsReaderPage = () => {
  const { content, onReady } = useTechDocsReader();

  if (!content) return null;

  return <TechDocsShadowDom source={content} onAttached={onReady} />;
};
```

This component uses `DOMPurify` as a sanitizer and you can configure it as well and add hooks for it to use (see the [DOMPurify](https://github.com/cure53/DOMPurify#dompurify) documentation for more on this):

```tsx
import {
  useTechDocsReader,
  TechDocsShadowDom,
  TechDocsShadowDomHooks,
} from '@backstage/plugin-techdocs';

import { beforeSanitizeElements, afterSanitizeAttributes } from './hooks';

export const CustomTechDocsReaderPage = () => {
  const configApi = useApi(configApiRef);
  const { content, onReady } = useTechDocsReader();

  if (!content) {
    return null;
  }

  const config = {
    ADD_TAGS: ['link'],
    FORBID_TAGS: ['style'],
  };

  const hooks: TechDocsShadowDomHooks = {
    afterSanitizeAttributes,
  };

  const sanitizer = configApi.getOptionalConfig('techdocs.sanitizer');
  const allowedHosts = sanitizer?.getOptionalStringArray('allowedIframeHosts');

  if (allowedHosts) {
    config.ADD_TAGS.push('iframe');
    hooks.beforeSanitizeElements = beforeSanitizeElements(allowedHosts);
  }

  return (
    <TechDocsShadowDom
      source={content}
      config={config}
      hooks={hooks}
      onAttached={onReady}
    />
  );
};
```

Note that the example above uses the Backstage APIs, you can use any api you implement in your Backstage instance!

Lastly, you might want to do some transformations on the dom element attached to the Shadow Root host and we encourage you to do this declaratively using React components, because these components can access the DOM via context by calling the `useTechDocsShadowRootDom` hook, encapsulate DOM manipulation inside a `useEffect`, benefit from calls from other hooks like for theme, apis and etc. and can return portals to elements inside the Shadow DOM.

```tsx
// transformers.tsx

import { useTechDocsShadowDom } from '@backstage/plugin-techdocs';
import { Portal } from '@material-ui/core';
import FeedbackOutlinedIcon from '@material-ui/icons/FeedbackOutlined';
import { FeedbackLink } from './FeedbackLink';

const EDIT_LINK_SELECTOR = '[title="Edit this page"]';

export const FeedbackLinkTransformer = () => {
  const dom = useTechDocsShadowDom();

  const editLink = dom.querySelector<HTMLAnchorElement>(EDIT_LINK_SELECTOR);

  if (!editLink) return null;

  let container = dom.querySelector('#git-feedback-link');

  if (!container) {
    container = document.createElement('div');
    container?.setAttribute('id', 'git-feedback-link');
    editLink.insertAdjacentElement('beforebegin', container);
  }

  return (
    <Portal container={container}>
      <a
        className="md-content__button md-icon"
        title="Leave feedback for this page"
        target="_blank"
        href="https://..."
        style={{ paddingLeft: '5px' }}
      >
        <FeedbackOutlinedIcon />
      </a>
    </Portal>

  return <FeedbackLink />;
};
```

```tsx
import {
  useTechDocsReader,
  TechDocsShadowDom,
  TechDocsShadowDomHooks,
} from '@backstage/plugin-techdocs';

import { FeedbackLinkTransformer } from './transformers';

export const CustomTechDocsReaderPage = () => {
  const configApi = useApi(configApiRef);
  const { content, onReady } = useTechDocsReader();

  if (!content) {
    return null;
  }

  return (
    <TechDocsShadowDom source={content} onAttached={onReady}>
      <FeedbackLinkTransformer />
    </TechDocsShadowDom>
  );
};
```
