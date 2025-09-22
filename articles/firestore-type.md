---
title: "Firestoreで適切に型をつけてデータを扱う方法を考えた（TypeScript/Next.js）"
emoji: "🔥"
type: "tech"
topics:
  - "firebase"
  - "firestore"
  - "typescript"
  - "nextjs"
published: true
published_at: "2023-02-26 15:24"
---

![](/images/firestore-type-hero.png)

フロントエンドは Next.js で、データは Firestore に保存する構成で軽く開発をしているが、TypeScript の型をいい感じにつけてデータを扱う方法を考えたのでメモする。

Firestore は NoSQL なのでどんな方のデータでも基本的に入れられてしまう。厳密にやるにはセキュリティ・ルールの方でも適切に制御する必要があるが、それはいったんおいておく。

データを操作する Next.js 側で型がある程度あると、データの取得や更新の際に型のチェックができるので、データの整合性を保つことができるはず。

## 最初に

Firestore にアクセスするためには Collection への参照をもつ必要がある。それを統一することで、特定の Collection に入るデータの型を定義できる。

```typescript
export const USERS_COLLECTION_KEY = "users";
export const getUsersCollection = (firestore: Firestore) => collection(firestore, USERS_COLLECTION_KEY);
```

こうすることで、`/users`の Collection を触るため使う Collection を使うことができる。

しかし、これだとまだ型の定義ができていない。Firestore は`converter`を設定することで、いい感じに型変換をしてくれるので順にやっていく。

## Firestore のデータを型を定義する

まずは、Firestore に保存するデータの型を定義する。すべてのデータ構造に共通して`createdAt`と`updatedAt`を持たせることにしたので、共通化するために`WithTimestamp`という型を定義している。

加えて、Firestore に保存するデータは`id`を持っていないが、Next.js 側で使うときには`id`がほしいので、snapshotId も含められるように`WithSnapshotId`という型を使って付与できるようにしている。

```typescript
// Firestore内のデータ構造
export type UserDocument = WithTimestamp<{
  name: string;
  email: string;
  profileImageRef?: string;
}>;
// フロントで使うデータ構造
export type UserDocumentWithId = WithSnapshotId<UserDocument>;
```

```typescript
// 別ファイル
export type WithTimestamp<T> = T & {
  createdAt?: Timestamp; // 作成中一瞬だけundefinedになる
  updatedAt?: Timestamp; // 更新中一瞬だけundefinedになる
};
export type WithSnapshotId<T> = T & { id: string };
```

## converter を定義する

converter では【**フロント → Firestore**】と【**Firestore → フロント**】の変換を定義する。

```typescript
export const userConverter: FirestoreDataConverter<UserDocumentWithId> = {
  toFirestore(doc): DocumentData {
    return {
      name: doc.name,
      email: doc.email,
      profileImageRef: doc.profileImageRef,
      createdAt: doc.createdAt,
      updatedAt: doc.updatedAt,
    };
  },

  fromFirestore(snapshot: QueryDocumentSnapshot<UserDocument>, options): UserDocumentWithId {
    const data = snapshot.data(options);
    const entity: UserDocumentWithId = {
      id: snapshot.id,
      name: data.name,
      email: data.email,
      profileImageRef: data.profileImageRef,
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
    };
    return entity;
  },
};
```

## collection に converter を設定する

converter を最初の collection を取得する部分に設定する。

```typescript
export const USERS_COLLECTION_KEY = "users";
export const getUsersCollection = (firestore: Firestore) => collection(firestore, USERS_COLLECTION_KEY).withConverter(userConverter);
```

`withConverter`を付ける前

![](/images/firestore-before-converter.png)

`withConverter`を付けた後

![](/images/firestore-after-converter.png)

ちゃんと`UserDocumentWithId`になっている。

## ファイル全体

まとめるとこんな感じ。

```typescript
import { DocumentData, Firestore, FirestoreDataConverter, QueryDocumentSnapshot, collection } from "firebase/firestore";

import { WithSnapshotId, WithTimestamp } from "~/types/firebase";

/**
 * Firestoreに保存するデータの型
 */
export type UserDocument = WithTimestamp<{
  name: string;
  email: string;
  profileImageRef?: string;
}>;

export type UserDocumentWithId = WithSnapshotId<UserDocument>;

export const USERS_COLLECTION_KEY = "users";
export const getUsersCollection = (firestore: Firestore) => collection(firestore, USERS_COLLECTION_KEY).withConverter(userConverter);

export const userConverter: FirestoreDataConverter<UserDocumentWithId> = {
  toFirestore(doc): DocumentData {
    return {
      name: doc.name,
      email: doc.email,
      profileImageRef: doc.profileImageRef,
      createdAt: doc.createdAt,
      updatedAt: doc.updatedAt,
    };
  },

  fromFirestore(snapshot: QueryDocumentSnapshot<UserDocument>, options): UserDocumentWithId {
    const data = snapshot.data(options);
    const entity: UserDocumentWithId = {
      id: snapshot.id,
      name: data.name,
      email: data.email,
      profileImageRef: data.profileImageRef,
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
    };
    return entity;
  },
};
```

## 最後に

あとは、この Collection への参照を使えば、必ずデータ構造が保証されることになるので、ある程度安全に使えるのではないかと思っている。

これから開発を進めて、また何か知見があればメモする。