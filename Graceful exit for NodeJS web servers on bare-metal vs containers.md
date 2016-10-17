### Motivation
Recently when experimenting with socket.io and express, I decided to use `docker-compose` to manage the environment. It made development much easier as I would simply make changes and run `docker-compose up` for sanity testing. But the longer I kept at it, the more obvious it became that the webserver was taking much longer to shut down in its containerized form than it did when I ran it directly on bare-metal with `node server.js`

### Trials and Observations
1. I spent some time rereading the differences in `CMD` vs. `ENTRYPOINT` for docker to see if the interrupt (`ctrl+c`) wasn't making it to the webserver inside the container. Switched over to using `ENTRYPOINT` but it didn't seem to make a difference in practice.
    1. The webserver container with only express would shutdown 10 to 20 seconds earlier than the webserver container with both express and socket.io
2. Active socket.io connections may not be disposed of even after receiving the interrupt so I rewrote the code to listen for various signals as outlined in this [blog](https://medium.com/@gchudnov/trapping-signals-in-docker-containers-7a57fdda7d86#.fjy8y81ru) to close the open websocket connections but it didn't seem to work either.
3. Decided that my code might be flawed so found libraries such as [http-shutdown](https://github.com/thedillonb/http-shutdown) and [express-graceful-exit](https://github.com/emostar/express-graceful-exit)
    1. `express-graceful-exit` seemed better as its documentation directly mentioned cleanup for socket.io
4. I started to suspect that the interrupts/signals weren't being received by nodejs processes so I added `console.log` statements which never showed up! I couldn't say if it was because `docker-compose` cuts off log aggregation upon `ctrl+c` or if the node processes were genuinely not receiving the interrupts.
5. I switched back to bare-metal with all the graceful exit code still intact and realized that `ctrl+c` would kill off the webserver with both express and socket.io instantly despite active websocket connections ... and there still weren't any logs generated from:
    ```
    process.on('SIGTERM', function() {
      console.log('shutting down...');
      ...
    });
    ```
    so I gave up! It seemed too much of a rabbit hole to troubleshoot at the moment but perhaps these notes will help myself or someone else (who finds it critical resolve this) to form a better experiment and deduct the correct approach.
