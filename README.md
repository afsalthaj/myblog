## Deployment of new posts

1. Go to posts/ and refer to existing ones, and create a new one similarly.
2. Build site

```scala
stack build

```
 
or

```scala

stack exec site clean
stack exec site build

```

3. Git push

```
git add -A
git commit -m "New post"
git push origin master

```

4. Goto https://afsalthaj.github.io/myblog/