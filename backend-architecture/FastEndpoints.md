# Fast Endpoints



To set up fast endpoints, we did the following:

On a new project, we aded the fast endpoints night package.

Then we added the security package, which includes JWT out of the box.





**Do we need cache?**

Maybe for big endpoints, like children lists



## Docker magic

To build an image:

```bash
$ docker build -t NAME:VERSION -f PATH/TO/DOCKERFILE .
```

To upload to Github containers:

First, we have to retag it:

```bash
$ docker tag NAME:VERSION ghcr.io/USERNAME/NAME:VERSION
```

Then login:

```bash
$  echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
```

Where CR_PAT is the access token for the GitHub Container Registry

Then we have to publish it:

```bash 
$ docker push ghcr.io/USERNAME/NAME:VERSION
```

