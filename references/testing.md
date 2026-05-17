# Testing with jest-stencil-runner

Package for Jest v30+: `jest-stencil-runner`
Import: `import { newSpecPage } from 'jest-stencil-runner'`

Tests live in `<topic>/tests/*.spec.ts` — separate from components and services.

---

## Testing Components — newSpecPage()

```ts
// chat/tests/chat-bubble.spec.ts
import { newSpecPage } from 'jest-stencil-runner';
import { ChatBubble } from '../components/chat-bubble/chat-bubble';

describe('chat-bubble', () => {

  it('renders with text', async () => {
    const { root } = await newSpecPage({
      components: [ChatBubble],
      html: '<chat-bubble text="Hallo Welt"></chat-bubble>',
    });

    expect(root).toEqualHtml(`
      <chat-bubble text="Hallo Welt">
        <div class="bubble">
          Hallo Welt
        </div>
      </chat-bubble>
    `);
  });

  it('has own class when own prop is set', async () => {
    const { root } = await newSpecPage({
      components: [ChatBubble],
      html: '<chat-bubble text="Hi" own></chat-bubble>',
    });

    expect(root.querySelector('.bubble')).toBeTruthy();
  });

  it('matches snapshot', async () => {
    const { root } = await newSpecPage({
      components: [ChatBubble],
      html: '<chat-bubble text="Snapshot Test"></chat-bubble>',
    });

    expect(root).toMatchSnapshot();
  });

});
```

---

## Testing Props & State

```ts
it('updates state when prop changes', async () => {
  const { root, waitForChanges } = await newSpecPage({
    components: [ChatBubble],
    html: '<chat-bubble text="Alt"></chat-bubble>',
  });

  // change prop
  root.setAttribute('text', 'Neu');
  await waitForChanges();

  expect(root.querySelector('.bubble').textContent).toBe('Neu');
});
```

---

## Testing Events

```ts
it('emits myChange event', async () => {
  const { root, waitForChanges } = await newSpecPage({
    components: [MyInput],
    html: '<my-input></my-input>',
  });

  const events: CustomEvent[] = [];
  root.addEventListener('myChange', (e: CustomEvent) => events.push(e));

  // trigger event
  root.click();
  await waitForChanges();

  const eventSpy = {
    events,
    firstEvent: events[0],
    lastEvent: events[events.length - 1],
    length: events.length,
  };

  expect(eventSpy).toHaveReceivedEvent();
  expect(eventSpy).toHaveReceivedEventTimes(1);
  expect(eventSpy).toHaveFirstReceivedEventDetail({ value: 'test' });
});
```

---

## Testing Services — plain unit tests

Services are plain TypeScript — no `newSpecPage()` needed, just instantiate directly.

```ts
// chat/tests/chat.service.spec.ts
import { ChatService, resetChatService } from '../services/chat.service';

describe('ChatService', () => {

  let service: ChatService;

  beforeEach(() => {
    resetChatService(); // reset singleton
    service = new ChatService();
  });

  it('returns empty list when no messages', () => {
    expect(service.getMessages()).toEqual([]);
  });

  it('adds a message', () => {
    service.addMessage({ text: 'Hello', from: 'user' });
    expect(service.getMessages()).toHaveLength(1);
  });

});
```

---

## Mocking Services in Component Tests

```ts
// replace service via jest.mock()
jest.mock('../../services/auth.service', () => ({
  getAuthService: () => ({
    isAuthenticated: () => true,
    getUser: () => ({ id: '1', email: 'test@test.de' }),
  }),
}));

import { newSpecPage } from 'jest-stencil-runner';
import { MyPage } from '../pages/my-page/my-page';

describe('my-page', () => {
  it('shows user email when logged in', async () => {
    const { root } = await newSpecPage({
      components: [MyPage],
      html: '<my-page></my-page>',
    });

    expect(root.textContent).toContain('test@test.de');
  });
});
```

> Always declare `jest.mock()` **before** importing the component under test.

---

## Custom Matchers Overview

| Matcher | Description |
|---------|-------------|
| `toEqualHtml(html)` | Compare HTML structure |
| `toMatchSnapshot()` | Create/compare snapshot |
| `toMatchInlineSnapshot()` | Inline snapshot |
| `toHaveReceivedEvent()` | Event was fired at least once |
| `toHaveReceivedEventTimes(n)` | Event was fired exactly n times |
| `toHaveFirstReceivedEventDetail(obj)` | Check detail of first event |
| `toHaveLastReceivedEventDetail(obj)` | Check detail of last event |

---

## Filename Convention

```
chat/
└── tests/
    ├── chat-bubble.spec.ts       ← component test
    ├── chat-input.spec.ts
    ├── chat.service.spec.ts      ← service test
    └── chat-detail-modal.spec.ts ← modal test
```

- Filename = tag name or service name + `.spec.ts`
- One `.spec.ts` per component/service
- All tests for a topic go in that topic's `tests/` folder
