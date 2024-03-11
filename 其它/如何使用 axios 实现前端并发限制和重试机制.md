# 如何使用 axios 实现前端并发限制和重试机制

在 Web 开发中，我们经常需要向后端发送多个异步请求以获取数据，然而过多的请求可能会对服务器造成过大的压力，影响系统的性能。因此，我们需要对并发请求进行限制。同时，由于网络环境的不稳定性，发送请求时也需要考虑添加重试机制，以提高请求的成功率和可靠性。

本篇博客将介绍如何使用 axios 实现前端并发限制和重试机制。axios 是一款基于 Promise 的 HTTP 客户端，可以用于浏览器和 Node.js 环境下发送 HTTP 请求。

## 前端并发限制的实现

我们可以使用一个请求队列来限制并发请求的数量，每次发送请求时，将请求加入到队列中，并检查当前队列的长度是否小于最大并发请求数量，如果是，就从队列中取出一个请求并发送；如果不是，就等待前面的请求完成后再发送下一个请求。

以下是一个使用 axios 实现前端并发限制的示例代码：

```javascript
const axios = require('axios');

// 最大并发请求数
const MAX_CONCURRENT_REQUESTS = 5;

// 请求队列
const requestQueue = [];
// 当前正在进行中的请求数
let activeRequests = 0;

/**
 * 处理请求队列中的请求
 */
function processQueue() {
  // 如果当前进行中的请求数没达到最大并发数 && 请求队列不为空
  if (activeRequests < MAX_CONCURRENT_REQUESTS && requestQueue.length > 0) {
    // 从请求队列中取出队头的请求
    const { url, config, resolve, reject } = requestQueue.shift();
    // 进行中的请求数 +1
    activeRequests++;
    // 通过 Axios 发送请求
    axios(url, config)
      .then((response) => {
        // 请求成功，将 外层 Promise 的状态更新为 fulfilled，并返回请求结果
        resolve(response);
      })
      .catch((error) => {
        // 请求失败，将 外层 Promise 的状态更新为 rejected，并返回错误信息
        reject(error);
      })
      .finally(() => {
        // 不论成功还是失败都会执行 finally，表示本次请求结束，将进行中的请求数 -1
        activeRequests--;
        // 再处理请求队列中的下一个请求
        processQueue();
      });
  }
}

/**
 * 并发请求方法
 * @param { string } url 请求的 url
 * @param { AxiosRequestConfig } config Axios 的请求配置
 */
function limitConcurrentRequests(url, config) { 
  // 这里很关键，将用户发起的每个请求都变成一个 Promise，而 Promise 的状态会在 processQueue 中根据 Axios 的执行结果来更新
  return new Promise((resolve, reject) => {
    // 将请求的配置信息 和 更新 Promise 状态的两个方法变成一个对象推入请求队列中
    requestQueue.push({ url, config, resolve, reject });
    // 执行 processQueue 方法处理请求队列中的每个请求
    processQueue();
  });
}

module.exports = { limitConcurrentRequests };

```

在以上代码中，我们设置了一个 `MAX_CONCURRENT_REQUESTS` 常量，表示最大并发请求数量，同时维护了一个请求队列 `requestQueue` 和一个变量 `activeRequests`，分别用于存储请求队列和正在处理的请求数量。我们通过定义 `processQueue` 方法来处理请求队列中的请求，它会检查当前正在处理的请求数量是否小于最大并发请求数量，并从队列中取出一个请求并发送。在发送请求的过程中，我们通过 Promise 的 `then` 和 `catch` 方法来处理成功和失败的情况，并在最后通过 `finally` 方法将正在处理的请求数量减一，并再次调用 `processQueue` 方法，以处理下一个请求。

使用时，我们可以通过以下代码来进行测试和梳理上述逻辑：

```javascript
const { limitConcurrentRequests } = require('./concurrency');

// 定义了 20个 URL
const urls = ['https://jsonplaceholder.typicode.com/posts/1', 'https://jsonplaceholder.typicode.com/posts/2', 'https://jsonplaceholder.typicode.com/posts/3', 'https://jsonplaceholder.typicode.com/posts/4', 'https://jsonplaceholder.typicode.com/posts/5', 'https://jsonplaceholder.typicode.com/posts/6', 'https://jsonplaceholder.typicode.com/posts/7', 'https://jsonplaceholder.typicode.com/posts/8', 'https://jsonplaceholder.typicode.com/posts/9', 'https://jsonplaceholder.typicode.com/posts/10', 'https://jsonplaceholder.typicode.com/posts/11', 'https://jsonplaceholder.typicode.com/posts/12', 'https://jsonplaceholder.typicode.com/posts/13', 'https://jsonplaceholder.typicode.com/posts/14', 'https://jsonplaceholder.typicode.com/posts/15', 'https://jsonplaceholder.typicode.com/posts/16', 'https://jsonplaceholder.typicode.com/posts/17', 'https://jsonplaceholder.typicode.com/posts/18', 'https://jsonplaceholder.typicode.com/posts/19', 'https://jsonplaceholder.typicode.com/posts/20'];

// 通过循环，同时发起 20 个请求
urls.forEach(url => limitConcurrentRequests(url)
  .then(responses => console.log(responses.data))
  .catch(error => console.error(error)));
```

在以上代码中，我们定义了一个包含 20 个 URL 的数组 `urls`，通过循环同时发送 20个请求。最后，我们通过 `then` 和 `catch` 方法分别处理请求成功和失败的情况，并打印出结果。

## 前端重试机制的实现

有时，由于网络环境的不稳定性，发送的请求可能会失败，因此我们需要对请求添加重试机制。在实现重试机制时，我们需要注意以下几点：

- 需要限制重试的次数，避免无限重试；
- 在重试过程中，需要添加一定的延迟时间，以避免过于频繁地发送请求；
- 重试时需要保证请求的幂等性，即多次重试的结果应该与单次请求的结果一致。

以下是一个使用 axios 实现前端重试机制的示例代码：

```javascript
const axios = require('axios');

// 最大重试次数，避免无限重试
const MAX_RETRY_TIMES = 3;
// 重试延时，避免频繁发送请求
const RETRY_DELAY = 1000;

/**
 * 支持重试机制的请求方法，整体方案是通过 Promise 包裹 Axios【这点和并发请求 limitConcurrentRequests 思路一样】 + 递归的逻辑来实现
 * @param { string } url 请求地址
 * @param { AxiosRequestConfig } config Axios 的请求配置
 * @param { number } retryTimes 请求最大重试次数
 * @returns Promise
 */
function retryableRequest(url, config, retryTimes = MAX_RETRY_TIMES) {
  return new Promise((resolve, reject) => {
    // 通过 Axios 发送请求
    axios(url, config)
      .then((response) => {
        // 请求成功，直接将 Promise 状态变为 fulfilled
        resolve(response);
      })
      .catch((error) => {
        // 请求失败
        if (retryTimes === 0) {
          // 剩余重试次数为 0，表示本次请求失败，将 Promise 状态从 pending 更新为 rejected
          reject(error);
        } else {
          // 还能继续重试，RETRY_DELAY 秒之后，递归调用 retryableRequest 方法，重新发送请求
          setTimeout(() => {
            // 递归逻辑，通过递归来实现重试，每次递归重试次数 -1；根据下层 retryableRequest 方法的 Promise 结果更新当前 Promise 的状态
            retryableRequest(url, config, retryTimes - 1)
              .then((response) => {
                // 请求成功，将 Promise 状态从 pending 更新为 fulfilled
                resolve(response);
              })
              .catch((error) => {
                // 请求失败，表示本次请求失败，将 Promise 状态从 pending 更新为 rejected
                reject(error);
              });
          }, RETRY_DELAY);
        }
      });
  });
}

module.exports = { retryableRequest };

```

在以上代码中，我们设置了一个 MAX_RETRY_TIMES 常量，表示最大重试次数，同时定义了一个 RETRY_DELAY 常量，表示重试的延迟时间。我们通过定义 retryableRequest 方法来实现重试机制，它会通过递归调用自身来重试请求，同时在每次重试前会添加一定的延迟时间以避免频繁发送请求。如果重试次数超过了最大重试次数，就会抛出错误。

以下是一个使用重试机制的示例代码：

```javascript
const { retryableRequest } = require('./retry');

retryableRequest('https://jsonplaceholder.typicode.com/posts/1', { method: 'get' })
  .then((response) => {
    console.log(response.data);
  })
  .catch((error) => {
    console.error(error);
  });

```

在以上代码中，我们使用了 retryableRequest 方法来发送请求，并在 then 和 catch 方法中处理请求成功和失败的情况。如果请求失败，重试机制会自动尝试重新发送请求，直到达到最大重试次数或请求成功为止。

## 并发限制 + 请求重试

上面分别讲述了 `前端并发限制` 和 `前端重试机制` 的实现，但两者逻辑独立，接下来会将两者结合，整体思路是在 并发限制 的基础上增加 请求重试。

```javascript
const axios = require('axios');

// 最大并发请求数
const MAX_CONCURRENT_REQUESTS = 5;

// 请求队列
const requestQueue = [];
// 当前正在进行中的请求数
let activeRequests = 0;

/**
 * 处理请求队列中的请求
 */
function processQueue() {
  // 如果当前进行中的请求数没达到最大并发数 && 请求队列不为空
  if (activeRequests < MAX_CONCURRENT_REQUESTS && requestQueue.length > 0) {
    // 从请求队列中取出队头的请求
    const { url, config, retryTimes, retryDelay, resolve, reject } = requestQueue.shift();
    // 进行中的请求数 +1
    activeRequests++;
    // 通过 Axios 发送请求
    axios(url, config)
      .then((response) => {
        // 请求成功，将 外层 Promise 的状态更新为 fulfilled，并返回请求结果
        resolve(response);
      })
      .catch((error) => {
        // 请求失败
        if (retryTimes === 0) {
          // 剩余重试次数为 0，表示本次请求失败，将 外层 Promise 的状态更新为 rejected，并返回错误信息
          reject(error);
        } else {
          // 还能继续重试，将请求重新入队
          setTimeout(() => {
            requestQueue.push({ url, config, retryTimes: retryTimes - 1, retryDelay, resolve, reject });
          }, retryDelay);
        }
      })
      .finally(() => {
        // 不论成功还是失败都会执行 finally，表示本次请求结束，将进行中的请求数 -1
        activeRequests--;
        // 再处理请求队列中的下一个请求
        processQueue();
      });
  }
}

/**
 * 请求方法
 * @param { string } url 请求的 url
 * @param { AxiosRequestConfig } config Axios 的请求配置
 * @param { number } retryTimes 最大重试次数，避免无限重试
 * @param { number } retryDelay 试延时，避免频繁发送请求
 */
function request(url, config, retryTimes = 3, retryDelay = 1000) { 
  // 这里很关键，将用户发起的每个请求都变成一个 Promise，而 Promise 的状态会在 processQueue 中根据 Axios 的执行结果来更新
  return new Promise((resolve, reject) => {
    // 将请求的配置信息 和 更新 Promise 状态的两个方法变成一个对象推入请求队列中
    requestQueue.push({ url, config, retryTimes, retryDelay, resolve, reject });
    // 执行 processQueue 方法处理请求队列中的每个请求
    processQueue();
  });
}

module.exports = { request };

```

## 总结

并发控制 + 请求重试整体思路还是比较简单的，Promise + 队列是关键。大家可以基于文中的代码扩充自己的业务逻辑。

这套思路可使用的场景有很多，比如 通过 refreshToken 刷新 token、URL 携带 ticket 实现免登录 等。

通过使用并发限制和重试机制，我们可以更好地控制前端请求的发送和处理。在实际开发中，我们需要根据具体的业务场景来选择合适的并发限制和重试机制，以确保请求的成功率和性能。
