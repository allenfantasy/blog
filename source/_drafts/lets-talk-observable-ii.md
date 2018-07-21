
### TOC

1. Subject
2. Observable from Scratch
3. Observable in Angular

====


5. 处理单个流的:
   * 简单的队列映射: `map`, `pluck`, `filter`, `scan`, `reduce`, `take`, `first`, `distinctUntilChanged` ...
   * 和时序有关的: `debounce`, `debounceTime`, `throttle`, `throttleTime`
   * 处理多个流之间关系的: `merge`, `concat`, `combineLatest`, `zip`, `withLatestFrom`
   * 降维的(源 observable 所释放的每个值又是一个 observable): `concatAll`, `mergeAll`, `combineAll`, `switch`
   * 映射+降维(源 observable 通过映射生成一个二维的 observable, 然后再降维): `concatMap`, `mergeMap`, `switchMap`
   * 其他: `every`, `defaultEmpty`, `sequenceEqual`, `delay` 等