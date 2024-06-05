# Nestjs Cache Manager for Service Method

## Usage Example

```ts
@Injectable()
export class LolVersionService {
  @CacheMethod()
  async getLatestVersion() {
    console.log('get version...');
    const { data } = await axios<string[]>(
      'https://ddragon.leagueoflegends.com/api/versions.json',
    );
    return data[0];
  }

  @CacheMethod({ ttlMs: 60 * 60 * 1000 })
  async someMethod(){
    // ...
  }
}
```

## Installation Steps

1. Install Package and Setup Module: Refer to the [NestJS Caching Documentation](https://docs.nestjs.com/techniques/caching) for installation and module setup instructions. 
2. Write CacheMethod Decorator: Utilize the provided CacheMethod decorator as demonstrated below.
```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { CACHE_KEY_METADATA, CACHE_TTL_METADATA, Inject } from '@nestjs/common';
import { Cache } from 'cache-manager';

export interface CacheOptions {
  ttlMs?: number;
}

/**
 * Note: Only work with args that can be serialized to JSON
 * @param options
 * @returns
 */
export function CacheMethod(
  options: CacheOptions = { ttlMs: 60000 },
): MethodDecorator {
  return (
    target: any,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor,
  ) => {
    // https://stackoverflow.com/a/75815629
    // https://stackoverflow.com/a/60608920

    const injector = Inject(CACHE_MANAGER);
    injector(target, CACHE_MANAGER);

    const originalMethod = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      const cacheKeyPrefix =
        Reflect.getMetadata(CACHE_KEY_METADATA, descriptor.value) ||
        Reflect.getMetadata(CACHE_KEY_METADATA, originalMethod) ||
        `${target.constructor.name}:${originalMethod.name}`;

      const cacheKey = `${cacheKeyPrefix}:${JSON.stringify(args)}`;

      const ttl =
        options.ttlMs ||
        Reflect.getMetadata(CACHE_TTL_METADATA, descriptor.value) ||
        Reflect.getMetadata(CACHE_TTL_METADATA, originalMethod) ||
        60000;

      const cacheManager = this[CACHE_MANAGER] as Cache;

      const cachedResult = await cacheManager.get(cacheKey);
      if (cachedResult) {
        return cachedResult;
      }

      const result = await originalMethod.apply(this, args);

      await cacheManager.set(cacheKey, result, ttl);

      return result;
    };

    return descriptor;
  };
}
```

Note: This code utilizes cache-manager v5.
