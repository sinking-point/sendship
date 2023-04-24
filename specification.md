# Sendship protocol specification

1. [Introduction](#introduction)
2. [Requirements](#requirements)
3. [Architecture Overview](#architecture-overview)
4. [Core Components](#core-components)
5. [API](#api)
6. [Security Considerations](#security-considerations)
7. [Privacy Considerations](#privacy-considerations)
8. [Implementation Guidelines](#implementation-guidelines)
9. [References](#references)

## Introduction

This document outlines the specification for the Sendship protocol.
The goal of this protocol is to create an open social network in which participants have full ownership of, and responsibility for, their own content and identity.
Each participant registers a domain name to serve as their identity.
They then host a server implementing Sendship on a subdomain (`sendship.example.com`) of that domain name.
The user logs in to their own server using a client, and follows other servers by specifying their domain names.
The servers communicate with each other by calling each other's API functions.
A user can publish their own posts, and browse posts from those they follow.
They can like, share and reply to others' posts, and send direct messages.

## Requirements

The requirements that the Sendship protocol should fulfil are as follows:

- Social networking features
- Decentralisation
- Identity ownership
- Content ownership
- Privacy
- Security
- Content safety

### Social networking features

An implementation of Sendgrid should enable a familiar social networking experience for the user.
This should include:

- Posts: Users can author and publish their own posts. A user's profile contains a timeline, showing their past posts.
- Followers: User A can follow user B.
- Feed: Then, B's posts appear on A's feed.
- Likes: Users can like posts. Posts keep a set of people who liked them.
- Shares: Users can share posts. When they share a post, it appears on their timeline, and on the feeds of their followers. Posts keep a list of links to shares. Shares are posts that contain a link to another post. They're intended audience is the sharer's followers.
- Replies: Users can reply to posts. Posts keep a list of links to replies. Replies are posts that contain a link to the parent. Their intended audience is the author and readers of the parent post.
- Direct messages: Users can send messages to each other privately.
- Multimedia: Posts can contain images and videos.

Note that a post can be both a share and a reply.

### Decentralisation

The protocol should assume that each server is self-contained.
The servers need not rely on any central actor for any purpose.
Exceptions apply outside the scope of the protocol, e.g. domain registrars and certificate authorities.

Implementors of the protocol are free to compromise on this as they see fit.
For example, it may be safer and more redundant to store data on a managed database, and multimedia files on AWS S3.

### Identity ownership

A flaw of previous social media platforms is that users lack ownership of their own identity.
If for any reason the user must leave the platform, they must also leave their identity on that platform behind.
They can make a new account with the same name on a different platform, but they must rebuild their following from nothing.

This applies even to platforms implementing the ActivityPub protocol (known as the Fediverse).
A user could join a Mastodon server, build up a large following, and then be banned at the server administrator's discretion.
Alternatively, the server could shut down.
The user then loses their identity and following.
This is because a webfinger address (e.g. `user@example.com`) includes the server domain.
This can be avoided by making what's known as a single-user instance, but that's not commonly done in the Fediverse.
The culture is to interact largely within one's own server, so single-user instances miss out on being part of a server community.

Sendship should solve this problem by using domain names as identities.
Once a user registers their own domain name, they own it.
The registrar should only revoke a domain name in extreme cases, such as hosting of illegal content.
This makes it unlikely for this identity to be stripped from a user at the arbitrary whim of another actor.

### Content ownership

As with identities, content is also vulnerable to deletion, suppression or modification on most social media platforms.
A user might spend hours writing a post only for it to be automatically deleted by an AI moderator for some imagined policy violation.
An algorithm might suppress a user's post, ensuring no one else gets to see it.
A Mastodon server administrator might decide to edit a user's posts for their own ends.

Sendship should make these impossible by keeping all of a user's content on the user's own personal server.

### Privacy

Sendship should give users fine-grained control over their privacy.
They should be able to decide who gets to see their content.
They should be able to block specific users.

### Security

Sendship needs 3 types of authentication:

1. Own server authenticates client (a user logs into their own Sendship server from their client)
2. Own server calls remote server (ensure we're talking to the genuine server for the domain name)
3. Remote server calls own server (ensure request comes from the other user it claims to come from)

Sendship needs secure encrypted communication both:

- Between client and server
- In server-to-server API calls

All API calls should check that the authenticated caller is authorised for the action it's attempting.

### Content safety

One of the challenges in a decentralised social network is protecting users from content they don't want to see.
Conventional centralised social media platforms accomplish this by moderating content, both manually and with AI tools.
This approach is not applicable in decentralised systems as there is no central authority.

The client design should allow users to prevent further exposure when they accidentally see disagreeable content.
Users should be able to easily block users who post disagreeable content.
Multimedia content should be easily hidden.

Users should be given the option to 'export to report' content.
This feature is intended to streamline the process of reporting illegal content to authorities such as police, domain registrars, ISPs etc..
This is necessary because only authorities such as these are capable of having the content taken down.
This should export the offending content to a file that the user can forward on to the appropriate authorities.
The file should contain all content associated with the reported post, including:

- The identity of the alleged offender
- The post text
- Any multimedia content
- Parent posts for context
- Cryptographic evidence that this content came from the alleged offender

## Architecture Overview

This section provides a high-level overview of the Sendship protocol's architecture, including its main components and their interactions.

### Sendship server

The Sendship server (or simply 'server') is a service that runs on an internet-connected, always-on machine.
It has 2 responsibilities:

1. Communicate with the client
2. Communicate with other servers

It's divided into 2 components:

1. API - This responds to web requests from both the client and other servers
2. Real-time notifications (RTN) - This uses sockets to do real-time bidirectional communication with the client for things like notifications and chat

### Sendship client

The Sendship client (or simply 'client') is an application that provides a user interface for interating with the server.
It has the following components:

1. Feed
2. Profile
3. Messages

## API

This section outlines the API used by the protocol.

### Error responses

Errors should use error codes appropriate for the cause of the error, and have a response body like this:

```json
    {
      "error": "string",
      "message": "string"
    }
```

### Create Post

This endpoint allows users to create a new post.

#### Authorization

Self only.

#### Request

**Endpoint:** `POST /api/v1/posts`

**Headers:**

- `Content-Type: application/json`
- `Authorization: Bearer {access_token}`

**Body:**

```json
{
  "content": "string",
  "images": [
    {
      "url": "string",
      "sha256": "string"
    }
  ] (optional),
  "videos": [
    {
      "url": "string",
      "sha256": "string"
    }
  ] (optional),
  "parent_post": {
    "domain": "string",
    "id": "string",
    "sha256": "string"
  } (optional),
  "shared_post": {
    "domain": "string",
    "id": "string",
    "sha256": "string"
  } (optional),
  "audience": "string"
}
```

#### Parameters

| Parameter   | Description                                                  | Type        | Required |
| ----------- | ------------------------------------------------------------ | ----------- | -------- |
| content     | The content of the post.                                     | string      | Yes      |
| images      | A list of image objects associated with the post.            | [Image]     | No       |
| videos      | A list of video objects associated with the post.            | [Video]     | No       |
| parent_post | An object containing information about the parent post.      | ParentPost  | No       |
| shared_post | An object containing information about the post being shared.| SharedPost  | No       |
| audience    | The visibility level of the post.                             | string      | Yes      |

##### Image Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| url       | The URL of the image.                  | string | Yes      |
| sha256    | The SHA-256 hash of the image content. | string | Yes      |

##### Video Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| url       | The URL of the video.                  | string | Yes      |
| sha256    | The SHA-256 hash of the video content. | string | Yes      |

##### Parent Post Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| domain    | The domain name of the parent post.     | string | Yes      |
| id        | The ID of the parent post.              | string | Yes      |
| sha256    | The SHA-256 hash of the parent post.    | string | Yes      |

##### Shared Post Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| domain    | The domain name of the shared post.     | string | Yes      |
| id        | The ID of the shared post.              | string | Yes      |
| sha256    | The SHA-256 hash of the shared post.    | string | Yes      |

##### Audience Values

| Value     | Description                                           |
| --------- | ----------------------------------------------------- |
| public    | Anyone can access the post.                            |
| sendship  | Any Sendship server can access the post.              |
| followees | Only the post creator and the people they follow can access. |

#### Response

**Status code:** `201 Created`

**Body:**

```json
{
  "id": "string"
}
```

### Get Post

This endpoint allows users to retrieve a post by its ID.

#### Authorization

Depends on the post's audience setting.

#### Request

**Endpoint:** `GET /api/v1/posts/{post_id}`

**Headers:**

- `Content-Type: application/json`
- `Authorization: Bearer {access_token}` (if required)

#### Path Parameters

| Parameter | Description                     | Type   | Required |
| --------- | ------------------------------- | ------ | -------- |
| post_id   | The unique identifier of a post | string | Yes      |

#### Response

**Status code:** `200 OK`

**Body:**

```json
{
  "id": "string",
  "created_at": "string",
  "updated_at": "string",
  "content": "string",
  "images": [
    {
      "url": "string",
      "sha256": "string"
    }
  ],
  "videos": [
    {
      "url": "string",
      "sha256": "string"
    }
  ],
  "parent_post": {
    "domain": "string",
    "id": "string",
    "sha256": "string"
  } (optional),
  "shared_post": {
    "domain": "string",
    "id": "string",
    "sha256": "string"
  } (optional),
  "audience": "string",
  "likes": [
    {
      "jwt": "string"
    }
  ],
  "shares": [
    {
      "jwt": "string"
    }
  ],
  "replies": [
    {
      "jwt": "string"
    }
  ]
}
```

#### Response Fields

| Field      | Description                                                  | Type        | Required |
| ---------- | ------------------------------------------------------------ | ----------- | -------- |
| id         | The unique identifier of the post.                           | string      | Yes      |
| created_at | The timestamp when the post was created (ISO 8601 format).   | string      | Yes      |
| updated_at | The timestamp when the post was last updated (ISO 8601 format). | string      | Yes      |
| content    | The textual content of the post.                                     | string      | Yes      |
| images     | A list of image objects associated with the post.            | [Image]     | No       |
| videos     | A list of video objects associated with the post.            | [Video]     | No       |
| parent_post | An object containing information about the parent post.      | ParentPost  | No       |
| shared_post | An object containing information about the post being shared.| SharedPost  | No       |
| audience    | The visibility level of the post.                             | string      | Yes      |
| likes       | A list of JWT strings representing likes on the post.                                                | [string]              | No       |
| shares      | A list of JWT strings representing shares of the post.                                               | [string]              | No       |
| replies     | A list of JWT strings representing replies to the post.                                              | [string]              | No       |

##### Image Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| url       | The URL of the image.                  | string | Yes      |
| sha256    | The SHA-256 hash of the image content. | string | Yes      |

##### Video Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| url       | The URL of the video.                  | string | Yes      |
| sha256    | The SHA-256 hash of the video content. | string | Yes      |

##### Parent Post Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| domain    | The domain name of the parent post.     | string | Yes      |
| id        | The ID of the parent post.              | string | Yes      |
| sha256    | The SHA-256 hash of the parent post.    | string | Yes      |

##### Shared Post Object

| Attribute | Description                             | Type   | Required |
| --------- | --------------------------------------- | ------ | -------- |
| domain    | The domain name of the shared post.     | string | Yes      |
| id        | The ID of the shared post.              | string | Yes      |
| sha256    | The SHA-256 hash of the shared post.    | string | Yes      |

##### Audience Values

| Value     | Description                                           |
| --------- | ----------------------------------------------------- |
| public    | Anyone can access the post. |
| sendship | Any Sendship server can access the post. |
| followees | Only the post creator and their followees can access. |

##### Likes

Each like in the likes list is a JWT string from the liker signed with RS256, containing the following claims:

| Claim        | Description                              | Type   |
| ------------ | ---------------------------------------- | ------ |
| liker_domain | The domain name of the liker.            | string |
| post_domain  | The domain name of the post.             | string |
| post_id      | The ID of the post being liked (should match this post's ID).          | string |

##### Shares

Each share in the shares list is a JWT string from the sharer signed with RS256, containing the following claims:

| Claim          | Description                                | Type   |
| -------------- | ------------------------------------------ | ------ |
| sharer_domain  | The domain name of the sharer.             | string |
| sharing_post_id| The ID of the post that shares the content.| string |
| shared_post_id | The ID of the post being shared (should match this post's ID).           | string |

##### Replies

Each reply in the replies list is a JWT string from the replier signed with RS256, containing the following claims:

| Claim          | Description                                      | Type   |
| -------------- | ------------------------------------------------ | ------ |
| replier_domain | The domain name of the replier.                  | string |
| reply_post_id  | The ID of the reply post.                        | string |
| parent_post_id | The ID of the post being replied to (should match this post's ID).             | string |

### TODO add more endpoints

## Security Considerations

TODO

### Threat 1

### Threat 2

### Threat 3

## Privacy Considerations

TODO

### Data Protection

### Privacy Risk 1

### Privacy Risk 2

## Implementation Guidelines

TODO

### Recommended Libraries and Frameworks

### Deployment Considerations

### Best Practices

## References

1. [Reference 1](https://example.com/reference1)
2. [Reference 2](https://example.com/reference2)
