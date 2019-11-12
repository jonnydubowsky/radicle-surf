# Design Documentation

In this document we will describe the design of `radicle-surf`. The design of the system will rely
heavily on [denotational design](todo) and use Haskell syntax (because types are easy to reason about, I'm sorry).

`radicle-surf` is a system to describe a file-system in a VCS world. We have the concept of files and directories,
but these objects can change over time while people iterate on them. Thus, it is a file-system within history and
we, the user, are viewing the file-system at a particular snapshot. Alongside this, we will wish to take two snapshots
and view their differences.

The stream of consciousness that gave birth to this document started with thinking how the user would interact with
the system, identifying the key components. This is captured in [User Flow](#user-flow). From there we found nouns that
represent objects in our system and verbs that represent functions over those objects. This iteratively informed us as
to what other actions we would need to supply. We would occassionally look at [GitHub](todo) and [Pijul Nest](todo) for
inspiration, since we would like to imitate the features that they supply, and we ultimately want use one or both of
these for our backends.

## User Flow

For the user flow we imagined what it would be like if the user was using a [REPL](todo) to interact with `radicle-surf`.
The general concept was that the user would enter the repository, build a view of the directory structure, and then
interact with the directories and files from there (called `browse`).
```haskell
repl :: IO ()
repl = do
  repo <- getRepo
  history <- getHistory label repo -- head is SHA1, tail is rest
  directory <- buildDirectory history

  forever browse directory
```

But then we thought about what happens when we are in `browse` but we would like to change the history and see that
file or directory at a different snapshot. This was captured in the pseudo-code below:
```haskell
  src_foo_bar <- find...
  history' <- historyOf src_foo_bar
```

This information was enough for us to begin the [denotational design](#denotational-design) below.

## Denotational Design

```haskell
-- A Label is a name for a directory or a file
type Label
μ Label = Text

-- A Directory captures its own Label followed by 1 or more
-- artefacts which can either be sub-directories or files.
--
-- An example of "foo/bar.hs" structure:
--  foo
--  |-- bar.hs
--
-- Would look like:
-- @("foo", Right ("bar.hs", "module Banana ...") :| [])@
type Directory
μ Directory = (Label, NonEmpty (Either Directory File))

-- A File is its Label and its contents
type File
μ File = (Label, ByteString)

-- An enumeration of what file-system artefact we're looking at.
-- Useful for listing a directory and denoting what the label is
-- corresponding to.
type SystemType
μ SystemType
  = IsFile
  | IsDirectory

-- A Chnage is an enumeration of how a file has changed.
-- This is simply used for getting the difference between two
-- directories.
type Change

-- Constructors of Change - think GADT
AddLineToFile :: NonEmpty Label -> Location -> ByteString -> Change
RemoveLineFromFile :: NonEmpty Label -> Location -> Change
MoveFile :: NonEmpty Label -> NonEmpty Label -> Change
CreateFile :: NonEmpty Label -> Change
DeleteFile :: NonEmpty Label -> Change

-- A Diff is a set of Changes that were made
type Diff
μ Diff = [Change]

-- History is an ordered set of @a@s. The reason for it being
-- polymorphic is that it allows us to choose what set artefact we
-- want to carry around.
--
-- For example:
--  * In `git` this would be a `Commit`.
--  * In `pijul` it would be a `Patch`.
type History a
μ History = [a]

-- A Repo is a collection of multiple histories.
-- This would essentially boil down to branches and tags.
type Repo
μ Repo a = [History a]

-- A Snapshot is a way of converting a History into a Directory.
-- In other words it gives us a snapshot of the history in the form of a directory.
type Snapshot a
μ Snapshot a = History a -> Directory

-- For example, we have a `git` snapshot or a `pjul` snapshot.
type Commit
type GitSnapshot   = Snapshot Commit

type Patch
type PijulSnapshot = Snapshot Patch

-- This is piece de resistance of the design! It turns out,
-- everything is just a Monad after all.
--
-- Our code Browser is a stateful computation of what History
-- we are currently working with and how to get a Snapshot of it.
type Browser a b
μ type Browser a b = ReaderT (Snapshot a) (State (History a) b)

-- A function that will retrieve a repository given an
-- identifier. In our case the identifier is opaque to the system.
getRepo :: Repo -> Repo

-- Find a particular History in the Repo. Again, how these things
-- are equated and found is opaque, but we can think of them as
-- branch or tag labels.
getHistory :: Eq a => History a -> Repo a -> Maybe (History a)
μ getHistory history repo =
  find history (μ repo)

-- Find if a particular artefact occurs in 0 or more histories.
findInHistories :: a -> [History a] -> [History a]
μ findInHistories a histories =
  filterMaybe (findInHistory a) histories

-- Find a particular artefact is in a history.
findInHistory :: Eq a => a -> History a -> Maybe a
μ findInHistory a history = find (== a) (μ history)

-- A special Label that guarantees a starting point, i.e. ~
root :: Label

-- Get the difference between two directory views.
diff :: Directory -> Directory -> Diff

-- List the current file or directories in a given Directory view.
listDirectory :: Directory -> NonEmpty (Label, SystemType)
μ listDirectory directory = map f $ snd (μ directory)
  where
    f = \case
      Left dir -> (fst dir, IsDirectory)
      Right file -> (fst file, IsFile)

fileName :: File -> Label
μ fileName file = fst (μ file)

findFile :: NonEmpty Label -> Directory -> Maybe File
μ findFile (label :| labels) directory =
  let (label, artefacts) = (μ directory)
  if label == label' then go labels artefacts else Nothing
  where
    findFileWithLabel :: Foldable f => Label -> f (Either Directory File) -> Maybe File
    findFileWithLabel label = find (\artefact -> case artefact of
      Left _     -> False
      Right file -> fileLabel == label)

    go :: [Label] -> [Either Directory File] -> Just File
    go [] _ = Nothing
    go [label] directories = findMaybe (fileWithLabel label) directories
    go (label:labels) directories = go labels $ find ((label ==) . fst) onlyDirectories directories

onlyDirectories :: Foldable f => f (Either Directory File) -> [Directory]
onlyDirectories = filter isLeft . toList

getSubDirectories :: Directory -> [Directory]
μ getSubDirectories directory = foldMap f $ snd (μ directory)
  where
    f :: Either Directory File -> [Directory]
    f = either pure []

-- Definition elided
findDirectory :: NonEmpty Label -> Directory -> Maybe Directory

-- Definition elided
fuzzyFind :: Label -> [Directory]

-- A Git Snapshot is grabbing the HEAD commit of your History
-- and turning it into a Directory
gitSnapshot :: Snapshot [Commit]
μ gitSnapshot = getDirectoryPtr . head

-- Opaque and defined by the backend
getDirectoryPtr :: Commit -> Directory

-- A Pijul history is semantically applying the patches in a
-- topological order and achieving the Directory view.
pijulHistory :: Snapshot Patch
μ pijulHistory = foldl pijulMagic mempty

-- Opaque and defined by the backend
pijulMagic :: Patch -> Directory -> Directory

-- Get the current History we are working with.
getHistory :: Browser a (History a)
μ getHistory = get

setHistory :: History a -> Browser a ()
μ setHistory = put

-- Get the current Directory in the Browser
getDirectory :: Browser a Directory
μ getDirectory = do
  hist <- get
  fromHistory <- ask
  pure $ fromHistory hist

-- We modify the history by changing the internal history state.
switchHistory :: ([a] -> [a]) -> Browser a b
μ switchHistory f = modify f

-- View the history up to a given point
viewAt :: Eq a => a -> Browser a b
μ viewAt a = switchHistory (dropWhile (/= a))
```