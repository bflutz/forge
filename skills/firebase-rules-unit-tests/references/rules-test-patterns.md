# Canonical Rules-Test Patterns

Copy-adapt these patterns. Each is a complete, runnable snippet against `@firebase/rules-unit-testing` >=4.0.

## Helper: `tests/rules/helpers.ts`

```ts
import {
  initializeTestEnvironment,
  RulesTestEnvironment,
  assertSucceeds,
  assertFails,
} from '@firebase/rules-unit-testing';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';

let cached: RulesTestEnvironment | null = null;

export async function getTestEnv(): Promise<RulesTestEnvironment> {
  if (cached) return cached;
  cached = await initializeTestEnvironment({
    projectId: 'demo-atlas',
    firestore: {
      rules: readFileSync(resolve(__dirname, '../../firestore.rules'), 'utf8'),
      host: '127.0.0.1',
      port: 9080,
    },
    storage: {
      rules: readFileSync(resolve(__dirname, '../../storage.rules'), 'utf8'),
      host: '127.0.0.1',
      port: 9199,
    },
  });
  return cached;
}

export async function seedUserDoc(
  env: RulesTestEnvironment,
  uid: string,
  role: 'viewer' | 'admin' | 'super-admin',
) {
  await env.withSecurityRulesDisabled(async (ctx) => {
    await ctx.firestore().doc(`users/${uid}`).set({ uid, role });
  });
}

export { assertSucceeds, assertFails };
```

Adjust `projectId` and ports per project. Ports come from `firebase-emulator-loop/references/emulator-ports.md`.

## Pattern 1: Authenticated read, admin write (catch-all)

```ts
import { describe, it, beforeEach, afterAll } from 'vitest';
import { getTestEnv, seedUserDoc, assertSucceeds, assertFails } from './helpers';
import { setDoc, getDoc, doc } from 'firebase/firestore';

describe('catch-all collection: alerts', () => {
  let env: Awaited<ReturnType<typeof getTestEnv>>;

  beforeEach(async () => {
    env = await getTestEnv();
    await env.clearFirestore();
    await seedUserDoc(env, 'admin-uid', 'admin');
    await seedUserDoc(env, 'viewer-uid', 'viewer');
    await env.withSecurityRulesDisabled(async (ctx) => {
      await setDoc(doc(ctx.firestore(), 'alerts/a1'), { title: 'seed' });
    });
  });

  afterAll(async () => env.cleanup());

  it('unauthenticated CANNOT read /alerts/a1', async () => {
    const db = env.unauthenticatedContext().firestore();
    await assertFails(getDoc(doc(db, 'alerts/a1')));
  });

  it('viewer CAN read /alerts/a1', async () => {
    const db = env.authenticatedContext('viewer-uid').firestore();
    await assertSucceeds(getDoc(doc(db, 'alerts/a1')));
  });

  it('viewer CANNOT write /alerts/a1', async () => {
    const db = env.authenticatedContext('viewer-uid').firestore();
    await assertFails(setDoc(doc(db, 'alerts/a1'), { title: 'hacked' }));
  });

  it('admin CAN write /alerts/a1', async () => {
    const db = env.authenticatedContext('admin-uid').firestore();
    await assertSucceeds(setDoc(doc(db, 'alerts/a1'), { title: 'updated' }));
  });
});
```

## Pattern 2: Owner-only access

```ts
describe('owner-only: notes', () => {
  beforeEach(async () => {
    await env.clearFirestore();
    await env.withSecurityRulesDisabled(async (ctx) => {
      await setDoc(doc(ctx.firestore(), 'notes/n1'), { ownerId: 'user-a' });
    });
  });

  it('owner CAN read own note', async () => {
    const db = env.authenticatedContext('user-a').firestore();
    await assertSucceeds(getDoc(doc(db, 'notes/n1')));
  });

  it('non-owner CANNOT read note', async () => {
    const db = env.authenticatedContext('user-b').firestore();
    await assertFails(getDoc(doc(db, 'notes/n1')));
  });

  it('owner CANNOT transfer ownership', async () => {
    const db = env.authenticatedContext('user-a').firestore();
    await assertFails(
      setDoc(doc(db, 'notes/n1'), { ownerId: 'user-b' }, { merge: true }),
    );
  });
});
```

Note: ownership-transfer test only works if the rule explicitly checks `request.resource.data.ownerId == resource.data.ownerId` or similar. If it doesn't, the test failure surfaces that the rule has a hole.

## Pattern 3: User creates own profile with restricted role

```ts
describe('users collection: self-create as viewer', () => {
  beforeEach(async () => env.clearFirestore());

  it('user CAN create own /users/{uid} with role=viewer', async () => {
    const db = env.authenticatedContext('new-user').firestore();
    await assertSucceeds(
      setDoc(doc(db, 'users/new-user'), {
        uid: 'new-user',
        role: 'viewer',
      }),
    );
  });

  it('user CANNOT create own /users/{uid} with role=admin (escalation)', async () => {
    const db = env.authenticatedContext('new-user').firestore();
    await assertFails(
      setDoc(doc(db, 'users/new-user'), {
        uid: 'new-user',
        role: 'admin',
      }),
    );
  });

  it('user CANNOT create another user profile', async () => {
    const db = env.authenticatedContext('attacker').firestore();
    await assertFails(
      setDoc(doc(db, 'users/victim'), { uid: 'victim', role: 'viewer' }),
    );
  });
});
```

## Pattern 4: Storage rules — fully-denied baseline

```ts
import { ref, uploadString, getDownloadURL } from 'firebase/storage';

describe('storage.rules: deny-all baseline', () => {
  it('authenticated CANNOT read', async () => {
    const storage = env.authenticatedContext('any').storage();
    await assertFails(getDownloadURL(ref(storage, 'any/path.txt')));
  });

  it('authenticated CANNOT write', async () => {
    const storage = env.authenticatedContext('any').storage();
    await assertFails(uploadString(ref(storage, 'any/path.txt'), 'data'));
  });
});
```

## Common gotchas

- **`assertFails` does not catch network errors.** If your test fails with "connect ECONNREFUSED", the emulator isn't running — that's environmental, not a rule deny.
- **`request.time` in rules.** If a rule depends on `request.time`, your test data may need explicit timestamps; use `Timestamp.now()` in seed data rather than server defaults.
- **List operations.** `getDocs(query(collection(db, 'users'), where(...)))` is a list operation; it needs `allow list` not `allow get`. Test list separately from get.
- **`request.resource` vs `resource`.** `request.resource.data` is the incoming write payload; `resource.data` is the existing document. Owner-transfer tests must use both.
