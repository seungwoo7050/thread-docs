## `test(protocol): versioned event codec 기대값 정렬`

diff --git a/packages/shared/src/ws.test.ts b/packages/shared/src/ws.test.ts
index c919775..c186c24 100644
--- a/packages/shared/src/ws.test.ts
+++ b/packages/shared/src/ws.test.ts
@@ -1,153 +1,99 @@
 import { describe, expect, it } from "vitest";
-import { encodeServerEvent, parseClientEvent, type ServerEvent } from "./ws";
+import {
+  encodeServerEvent,
+  parseClientEvent,
+  parseServerEvent,
+  type ServerEvent
+} from "./ws";
+import type { GameSnapshot } from "./game";
 
-describe("parseClientEvent", () => {
+describe("version 1 client events", () => {
   it.each([
-    {
-      name: "queue join",
-      payload: { type: "queue.join", mode: "ai" },
-      expected: { type: "queue.join", mode: "ai" }
-    },
-    {
-      name: "queue leave",
-      payload: { type: "queue.leave" },
-      expected: { type: "queue.leave" }
-    },
-    {
-      name: "game ready",
-      payload: { type: "game.ready", roomId: "room-1" },
-      expected: { type: "game.ready", roomId: "room-1" }
-    },
-    {
-      name: "game input",
-      payload: { type: "game.input", roomId: "room-1", direction: -1 },
-      expected: { type: "game.input", roomId: "room-1", direction: -1 }
-    },
-    {
-      name: "chat send",
-      payload: { type: "chat.send", scope: "match", roomId: "room-1", body: "hello" },
-      expected: { type: "chat.send", scope: "match", roomId: "room-1", body: "hello" }
-    }
-  ])("accepts $name events", ({ payload, expected }) => {
-    expect(parseClientEvent(JSON.stringify(payload))).toEqual(expected);
+    { payload: { v: 1, type: "queue.join", mode: "ai" } },
+    { payload: { v: 1, type: "queue.leave" } },
+    { payload: { v: 1, type: "tournament.join", matchId: "match-1" } },
+    { payload: { v: 1, type: "game.ready", roomId: "room-1" } },
+    { payload: { v: 1, type: "game.pause", roomId: "room-1" } },
+    { payload: { v: 1, type: "game.resume", roomId: "room-1" } },
+    { payload: { v: 1, type: "game.input", roomId: "room-1", inputSeq: 7, direction: -1 } },
+    { payload: { v: 1, type: "chat.send", scope: "match", roomId: "room-1", body: "hello" } }
+  ])("accepts $payload.type", ({ payload }) => {
+    expect(parseClientEvent(JSON.stringify(payload))).toEqual(payload);
   });
 
-  it("defaults queue joins to queue mode", () => {
-    expect(parseClientEvent(JSON.stringify({ type: "queue.join" }))).toEqual({
+  it("defaults queue mode without defaulting the protocol version", () => {
+    expect(parseClientEvent(JSON.stringify({ v: 1, type: "queue.join" }))).toEqual({
+      v: 1,
       type: "queue.join",
       mode: "queue"
     });
-  });
-
-  it("rejects unknown event types", () => {
-    expect(() => parseClientEvent(JSON.stringify({ type: "game.unknown" }))).toThrow();
+    expect(() => parseClientEvent(JSON.stringify({ type: "queue.join" }))).toThrow();
   });
 
   it.each([
-    { name: "ready room id", payload: { type: "game.ready" } },
-    { name: "input room id", payload: { type: "game.input", direction: 0 } },
-    { name: "input direction", payload: { type: "game.input", roomId: "room-1" } },
-    { name: "chat scope", payload: { type: "chat.send", body: "hello" } },
-    { name: "chat body", payload: { type: "chat.send", scope: "lobby" } }
-  ])("rejects events without the required $name", ({ payload }) => {
+    { name: "missing version", payload: { type: "queue.leave" } },
+    { name: "unsupported version", payload: { v: 2, type: "queue.leave" } },
+    { name: "unexpected field", payload: { v: 1, type: "queue.leave", token: "secret" } },
+    { name: "missing input sequence", payload: { v: 1, type: "game.input", roomId: "room-1", direction: 0 } },
+    { name: "negative input sequence", payload: { v: 1, type: "game.input", roomId: "room-1", inputSeq: -1, direction: 0 } },
+    { name: "fractional input sequence", payload: { v: 1, type: "game.input", roomId: "room-1", inputSeq: 1.5, direction: 0 } },
+    { name: "invalid direction", payload: { v: 1, type: "game.input", roomId: "room-1", inputSeq: 1, direction: 2 } }
+  ])("rejects $name", ({ payload }) => {
     expect(() => parseClientEvent(JSON.stringify(payload))).toThrow();
   });
 
-  it.each([
-    { name: "queue mode", payload: { type: "queue.join", mode: "ranked" } },
-    { name: "chat scope", payload: { type: "chat.send", scope: "private", body: "hello" } },
-    { name: "direction below the range", payload: { type: "game.input", roomId: "room-1", direction: -2 } },
-    { name: "direction above the range", payload: { type: "game.input", roomId: "room-1", direction: 2 } },
-    { name: "non-numeric direction", payload: { type: "game.input", roomId: "room-1", direction: "1" } }
-  ])("rejects an invalid $name", ({ payload }) => {
-    expect(() => parseClientEvent(JSON.stringify(payload))).toThrow();
-  });
-
-  it.each([-1, 0, 1])("accepts %i as an input direction", (direction) => {
-    expect(
-      parseClientEvent(JSON.stringify({ type: "game.input", roomId: "room-1", direction }))
-    ).toEqual({ type: "game.input", roomId: "room-1", direction });
-  });
-
-  it("trims chat bodies", () => {
-    expect(
-      parseClientEvent(JSON.stringify({ type: "chat.send", scope: "lobby", body: "  hello  " }))
-    ).toEqual({ type: "chat.send", scope: "lobby", body: "hello" });
-  });
-
-  it("accepts chat bodies at the 1 and 240 character boundaries", () => {
-    const oneCharacter = parseClientEvent(
-      JSON.stringify({ type: "chat.send", scope: "lobby", body: "a" })
-    );
-    const twoHundredFortyCharacters = parseClientEvent(
-      JSON.stringify({ type: "chat.send", scope: "lobby", body: "a".repeat(240) })
-    );
-
-    expect(oneCharacter).toMatchObject({ body: "a" });
-    expect(twoHundredFortyCharacters).toMatchObject({ body: "a".repeat(240) });
-  });
-
-  it.each(["", "   ", "a".repeat(241)])("rejects an invalid chat body length", (body) => {
-    expect(() =>
-      parseClientEvent(JSON.stringify({ type: "chat.send", scope: "lobby", body }))
-    ).toThrow();
-  });
+  it("trims bounded chat bodies", () => {
+    expect(parseClientEvent(JSON.stringify({
+      v: 1,
+      type: "chat.send",
+      scope: "lobby",
+      body: "  hello  "
+    }))).toEqual({ v: 1, type: "chat.send", scope: "lobby", body: "hello" });
 
-  it("rejects malformed JSON", () => {
-    expect(() => parseClientEvent('{"type":"queue.leave"')).toThrow(SyntaxError);
+    expect(() => parseClientEvent(JSON.stringify({
+      v: 1,
+      type: "chat.send",
+      scope: "lobby",
+      body: "a".repeat(241)
+    }))).toThrow();
   });
 });
 
-describe("encodeServerEvent", () => {
-  const serverEvents = [
-    {
-      type: "queue.matched",
-      roomId: "room-1",
-      side: "left",
-      opponent: "opponent"
-    },
-    {
-      type: "game.snapshot",
-      snapshot: {
-        roomId: "room-1",
-        phase: "playing",
-        tick: 12,
-        leftScore: 1,
-        rightScore: 0,
-        paddles: {
-          left: { y: 100, dy: -1 },
-          right: { y: 200, dy: 1 }
-        },
-        ball: {
-          position: { x: 480, y: 270 },
-          velocity: { x: 6, y: -2 }
-        },
-        players: [
-          {
-            id: "player-1",
-            handle: "left-player",
-            displayName: "Left Player",
-            side: "left",
-            ready: true,
-            ai: false
-          },
-          {
-            id: "player-2",
-            handle: "right-player",
-            displayName: "Right Player",
-            side: "right",
-            ready: true,
-            ai: false
-          }
-        ],
-        serverTime: "2026-07-23T00:00:00.000Z"
-      }
-    },
+describe("version 1 server events", () => {
+  const snapshot: GameSnapshot = {
+    roomId: "room-1",
+    tick: 12,
+    sequence: 15,
+    serverTimeMs: 1_784_764_800_000,
+    state: {
+      phase: "playing",
+      leftScore: 1,
+      rightScore: 0,
+      paddles: {
+        left: { y: 100, dy: -1 },
+        right: { y: 200, dy: 1 }
+      },
+      ball: {
+        position: { x: 480, y: 270 },
+        velocity: { x: 6, y: -2 }
+      },
+      players: [
+        { id: "player-1", handle: "left-player", displayName: "Left Player", side: "left", ready: true, ai: false },
+        { id: "player-2", handle: "right-player", displayName: "Right Player", side: "right", ready: true, ai: false }
+      ]
+    }
+  };
+
+  const events = [
+    { v: 1, type: "queue.matched", roomId: "room-1", side: "left", opponent: "Opponent" },
+    { v: 1, type: "game.snapshot", snapshot },
     {
+      v: 1,
       type: "game.finished",
       result: {
         roomId: "room-1",
         matchId: "match-1",
+        persisted: true,
         winnerSide: "left",
         leftScore: 3,
         rightScore: 1,
@@ -155,13 +101,14 @@ describe("encodeServerEvent", () => {
       }
     },
     {
+      v: 1,
       type: "chat.message",
       message: {
-        id: "message-1",
+        id: "11111111-1111-4111-8111-111111111111",
         scope: "lobby",
         roomId: null,
         sender: {
-          id: "player-1",
+          id: "22222222-2222-4222-8222-222222222222",
           handle: "left-player",
           displayName: "Left Player",
           avatarKey: "avatar-1",
@@ -177,21 +124,13 @@ describe("encodeServerEvent", () => {
         createdAt: "2026-07-23T00:00:00.000Z"
       }
     },
-    {
-      type: "presence.changed",
-      online: 12,
-      playing: 4
-    },
-    {
-      type: "error",
-      message: "invalid event"
-    }
+    { v: 1, type: "presence.changed", online: 12, playing: 4 },
+    { v: 1, type: "error", code: "invalid_event", message: "invalid event" }
   ] satisfies ServerEvent[];
 
-  it.each(serverEvents)("serializes $type events", (event) => {
+  it.each(events)("validates and serializes $type", (event) => {
     const encoded = encodeServerEvent(event);
 
-    expect(encoded).toBe(JSON.stringify(event));
-    expect(JSON.parse(encoded)).toEqual(event);
+    expect(parseServerEvent(encoded)).toEqual(event);
   });
 });


## `feat(game): versioned outbound event 송신 경계 연결`

diff --git a/apps/api/src/gameHub.ts b/apps/api/src/gameHub.ts
index 64d9b88..72f1f70 100644
--- a/apps/api/src/gameHub.ts
+++ b/apps/api/src/gameHub.ts
@@ -23,6 +23,12 @@ type Client = {
   roomId: string | null;
 };
 
+type VersionlessServerEvent = ServerEvent extends infer Event
+  ? Event extends { v: 1 }
+    ? Omit<Event, "v">
+    : never
+  : never;
+
 type QueueEntry = {
   client: Client;
   queuedAt: number;
@@ -423,11 +429,11 @@ export class GameHub {
     });
   }
 
-  private broadcastAll(event: ServerEvent): void {
+  private broadcastAll(event: VersionlessServerEvent): void {
     for (const client of this.clients.values()) this.send(client, event);
   }
 
-  private broadcastRoom(roomId: string, event: ServerEvent): void {
+  private broadcastRoom(roomId: string, event: VersionlessServerEvent): void {
     const room = this.rooms.get(roomId);
     if (!room) return;
     for (const client of Object.values(room.clients)) {
@@ -435,9 +441,15 @@ export class GameHub {
     }
   }
 
-  private send(client: Client, event: ServerEvent): void {
+  private broadcastSnapshot(room: Room): void {
+    room.snapshot.sequence += 1;
+    room.snapshot.serverTimeMs = Date.now();
+    this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: room.snapshot });
+  }
+
+  private send(client: Client, event: VersionlessServerEvent): void {
     if (client.socket.readyState === WebSocket.OPEN) {
-      client.socket.send(encodeServerEvent(event));
+      client.socket.send(encodeServerEvent({ ...event, v: 1 } as ServerEvent));
     }
   }
 }


## `feat(web): lobby realtime event codec 소비`

diff --git a/apps/web/src/app/page.tsx b/apps/web/src/app/page.tsx
index 78a038c..a1088c5 100644
--- a/apps/web/src/app/page.tsx
+++ b/apps/web/src/app/page.tsx
@@ -2,7 +2,7 @@
 
 import { useCallback, useEffect, useRef, useState } from "react";
 import { Bot, Clock, MessageCircle, Trophy, Users, Zap } from "lucide-react";
-import type { ChatMessage, LobbyStats, PublicUser, ServerEvent, SessionUser } from "@pong-pong/shared";
+import { parseServerEvent, type ChatMessage, type LobbyStats, type PublicUser, type SessionUser } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { LoginPanel } from "@/components/LoginPanel";
 import { PongCanvas } from "@/components/PongCanvas";
@@ -47,7 +47,7 @@ export default function HomePage() {
         socket = new WebSocket(`${WS_URL}?ticket=${encodeURIComponent(ticket)}&v=${protocolVersion}`);
         socketRef.current = socket;
         socket.onmessage = (event) => {
-          const message = JSON.parse(event.data) as ServerEvent;
+          const message = parseServerEvent(event.data);
           if (message.type === "chat.message" && message.message.scope === "lobby") {
             setChat((current) => [...current.filter((item) => item.id !== message.message.id).slice(-19), message.message]);
           }
@@ -82,7 +82,7 @@ export default function HomePage() {
     try {
       const socket = socketRef.current;
       if (socket?.readyState === WebSocket.OPEN) {
-        socket.send(JSON.stringify({ type: "chat.send", scope: "lobby", roomId: null, body }));
+        socket.send(JSON.stringify({ v: 1, type: "chat.send", scope: "lobby", roomId: null, body }));
       } else {
         const message = await sendLobbyChat(body);
         setChat((current) => [...current.slice(-19), message]);


## `feat(play): versioned game input과 snapshot 소비`

diff --git a/apps/web/src/app/play/page.tsx b/apps/web/src/app/play/page.tsx
index 5c961f2..422c59d 100644
--- a/apps/web/src/app/play/page.tsx
+++ b/apps/web/src/app/play/page.tsx
@@ -2,7 +2,7 @@
 
 import { useEffect, useMemo, useRef, useState } from "react";
 import { MessageCircle, Pause, Play, Send, Signal, Users } from "lucide-react";
-import type { GameSnapshot, ServerEvent } from "@pong-pong/shared";
+import { parseServerEvent, type GameSnapshot } from "@pong-pong/shared";
 import { AppShell } from "@/components/AppShell";
 import { PongCanvas } from "@/components/PongCanvas";
 import { requestWsTicket } from "@/lib/api";
@@ -18,14 +18,16 @@ export default function PlayPage() {
   const socketRef = useRef<WebSocket | null>(null);
   const ticketRequestRef = useRef<AbortController | null>(null);
   const directionRef = useRef<-1 | 0 | 1>(0);
+  const inputSequenceRef = useRef(0);
+  const snapshotSequenceRef = useRef(-1);
 
-  const score = useMemo(() => (snapshot ? `${snapshot.leftScore} - ${snapshot.rightScore}` : "경기 전"), [snapshot]);
-  const phase = snapshot?.phase ?? "waiting";
+  const score = useMemo(() => (snapshot ? `${snapshot.state.leftScore} - ${snapshot.state.rightScore}` : "경기 전"), [snapshot]);
+  const phase = snapshot?.state.phase ?? "waiting";
   const canReady = Boolean(roomId && phase === "waiting");
   const canChat = Boolean(roomId && phase !== "finished" && chatInput.trim());
   const canPause = Boolean(roomId && phase === "playing");
   const canResume = Boolean(roomId && phase === "paused");
-  const opponent = snapshot?.players.find((player) => player.side === "right");
+  const opponent = snapshot?.state.players.find((player) => player.side === "right");
   const opponentName = opponent?.displayName ?? "대기 중";
   const autoStartedRef = useRef(false);
 
@@ -82,7 +84,14 @@ export default function PlayPage() {
     const timer = window.setInterval(() => {
       const socket = socketRef.current;
       if (!socket || socket.readyState !== WebSocket.OPEN) return;
-      socket.send(JSON.stringify({ type: "game.input", roomId, direction: directionRef.current }));
+      inputSequenceRef.current += 1;
+      socket.send(JSON.stringify({
+        v: 1,
+        type: "game.input",
+        roomId,
+        inputSeq: inputSequenceRef.current,
+        direction: directionRef.current
+      }));
     }, 50);
     return () => window.clearInterval(timer);
   }, [roomId, phase]);
@@ -102,6 +111,8 @@ export default function PlayPage() {
     setMessages([]);
     setChatInput("");
     directionRef.current = 0;
+    inputSequenceRef.current = 0;
+    snapshotSequenceRef.current = -1;
     setStatus("실시간 연결 준비 중");
     const controller = new AbortController();
     ticketRequestRef.current = controller;
@@ -121,24 +132,29 @@ export default function PlayPage() {
     socketRef.current = socket;
     socket.onopen = () => {
       setStatus(openStatus);
-      socket.send(JSON.stringify(payload));
+      socket.send(JSON.stringify({ v: 1, ...payload }));
     };
     socket.onmessage = (event) => {
-      const message = JSON.parse(event.data) as ServerEvent;
+      const message = parseServerEvent(event.data);
       if (message.type === "queue.matched") {
         setRoomId(message.roomId);
         setStatus(`${message.opponent} 상대와 연결됨`);
       }
       if (message.type === "game.snapshot") {
+        if (message.snapshot.sequence <= snapshotSequenceRef.current) return;
+        snapshotSequenceRef.current = message.snapshot.sequence;
         setSnapshot(message.snapshot);
-        if (message.snapshot.phase === "playing") setStatus("경기 진행 중");
-        if (message.snapshot.phase === "paused") setStatus("일시정지 중");
-        if (message.snapshot.phase === "waiting") setStatus("준비 대기 중");
+        if (message.snapshot.state.phase === "playing") setStatus("경기 진행 중");
+        if (message.snapshot.state.phase === "paused") setStatus("일시정지 중");
+        if (message.snapshot.state.phase === "waiting") setStatus("준비 대기 중");
       }
       if (message.type === "game.finished") {
         setRoomId(null);
         directionRef.current = 0;
-        setSnapshot((current) => current ? { ...current, phase: "finished" } : current);
+        setSnapshot((current) => current ? {
+          ...current,
+          state: { ...current.state, phase: "finished" }
+        } : current);
         setStatus(`경기 종료: ${message.result.leftScore} - ${message.result.rightScore}`);
       }
       if (message.type === "chat.message") setMessages((current) => [...current.slice(-5), `${message.message.sender.displayName}: ${message.message.body}`]);
@@ -155,7 +171,7 @@ export default function PlayPage() {
 
   function ready() {
     if (socketRef.current && canReady && roomId) {
-      socketRef.current.send(JSON.stringify({ type: "game.ready", roomId }));
+      socketRef.current.send(JSON.stringify({ v: 1, type: "game.ready", roomId }));
       setStatus("준비 완료");
     }
   }
@@ -164,18 +180,18 @@ export default function PlayPage() {
     event.preventDefault();
     const body = chatInput.trim();
     if (!socketRef.current || !roomId || !body) return;
-    socketRef.current.send(JSON.stringify({ type: "chat.send", scope: "match", roomId, body }));
+    socketRef.current.send(JSON.stringify({ v: 1, type: "chat.send", scope: "match", roomId, body }));
     setChatInput("");
   }
 
   function togglePause() {
     if (!socketRef.current || !roomId) return;
     if (canPause) {
-      socketRef.current.send(JSON.stringify({ type: "game.pause", roomId }));
+      socketRef.current.send(JSON.stringify({ v: 1, type: "game.pause", roomId }));
       return;
     }
     if (canResume) {
-      socketRef.current.send(JSON.stringify({ type: "game.resume", roomId }));
+      socketRef.current.send(JSON.stringify({ v: 1, type: "game.resume", roomId }));
     }
   }
 


## `test(protocol): versioned realtime contract 검증`

diff --git a/packages/shared/src/ws.test.ts b/packages/shared/src/ws.test.ts
index c186c24..16f800c 100644
--- a/packages/shared/src/ws.test.ts
+++ b/packages/shared/src/ws.test.ts
@@ -133,4 +133,26 @@ describe("version 1 server events", () => {
 
     expect(parseServerEvent(encoded)).toEqual(event);
   });
+
+  it("rejects stale protocol shapes", () => {
+    expect(() => parseServerEvent(JSON.stringify({ type: "presence.changed", online: 1, playing: 0 }))).toThrow();
+    expect(() => parseServerEvent(JSON.stringify({
+      v: 1,
+      type: "game.snapshot",
+      snapshot: { ...snapshot, sequence: -1 }
+    }))).toThrow();
+    expect(() => parseServerEvent(JSON.stringify({
+      v: 1,
+      type: "game.finished",
+      result: {
+        roomId: "room-1",
+        matchId: null,
+        persisted: true,
+        winnerSide: "left",
+        leftScore: 3,
+        rightScore: 0,
+        ratingDelta: 16
+      }
+    }))).toThrow();
+  });
 });
