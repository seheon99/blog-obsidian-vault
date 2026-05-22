---
title:
type: til
topics:
description:
createdAt:
---
## `__init__.py`

## Connection, Cursor
1 request - 1 connection - N cursors
singleton connection

## Depends, `yield`

> [!NOTE]
> 그럼 다시 돌아와서, FastAPI 에서 한 request 마다 하나의 connection 을 만들려면 어떻게해? 한 요청이 여러 서비스에서 repository 호출을 할 수 있잖아. connection 생성을 repository 에 넣으면 한 요청마다 여러 커넥션이 만들어질 수도 있고, connection 생성을 router 단에 넣으면 매 repository 호출마다 connection 을 들고 다녀야해서 props drilling 같은 문제가 발생할탠데

> [!INFO]
> 한 요청에서 여러 번 필요하면 기본적으로 한 번만 호출하고 그 값을 캐시해서 재사용함
> `Denepds(..., use_cache=True)` 가 기본값

## Dependency Injection

> [!note]
> 객체가 필요한 의존성을 바깥에서 만들어 넣어준다

