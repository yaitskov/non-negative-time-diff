# non-negative-time-diff

Both arguments of `diffUTCTime` function from `time` package have the
same type.  It is easy to mix them.

``` haskell
f = do
  started <- getCurrentTime
  threadDelay 10_000_000
  ended <- getCurrentTime
  pure $ started `diffUTCTime` ended
```

This package provides a stricter `diffUTCTime` that significantly
reduces possibility of mixing its arguments by an accident.

``` haskell
import Data.Time.Clock.NonNegativeTimeDiff
f = do
  started <- getCurrentTime
  threadDelay 10_000_000
  ended <- getTimeAfter started
  pure $ ended `diffUTCTime` started
```

## STM use case
The STM package is shipped without a function to get current time.
Let's consider a situtation like this:

``` haskell
data Ctx
  = Ctx { m :: Map Int UTCTime
        , s :: TVar NominalDiffTime
        , q :: TQueue Int
        }

f (c :: Ctx) = do
  now <- getCurrentTime
  atomically $ do
    i <- readTQueue q
    lookup i c.m >>= \case
      Nothing -> pure ()
      Just t -> modifyTVar' c.s (+ diffUTCTime now t)
```

`now` might be less than `t` because the queue might be empty by the
time `f` is invoked.
The package API can correct the above snippet as follows:

``` haskell
data Ctx
  = Ctx { m :: Map Int UtcBox
        , s :: TVar NominalDiffTime
        , q :: TQueue Int
        }

f (c :: Ctx) = do
  atomically $ do
    i <- readTQueue q
    lookup i c.m >>= \case
      Nothing -> pure ()
      Just t ->
        doAfter tb \t -> do
          now <- getTimeAfter t
          modifyTVar' c.s (+ diffUTCTime now t)
```

## File access time

Another popular usecase where original `diffUTCTime` might be misused.

``` haskell
isFileOlderThan :: FilePath -> NominalDiffTime -> IO Bool
isFileOlderThan fp maxAge = do
  now <- getCurrentTime
  mt <- getModificationTime fp
  when (mt `diffUTCTime` now > maxAge) $ do
    removeFile fp
```

File age is always negative in the above example - this eventually
would cause a space leak on disk.

Corrected version:
``` haskell
isFileOlderThan :: FilePath -> NominalDiffTime -> IO Bool
isFileOlderThan fp maxAge =
  getModificationTime fp >>= (`doAfter` \mt -> do
    now <- getTimeAfter mt
    when (now `diffUTCTime` mt > maxAge) $ do
      removeFile fp)
```

## Requirements

Unboxing `UtcBox` values requires a GHC [natnormalise
plugin](https://hackage.haskell.org/package/ghc-typelits-natnormalise):

``` haskell
{-# GHC_OPTIONS -fplugin GHC.TypeLits.Normalise #-}

```
