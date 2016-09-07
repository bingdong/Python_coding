# tookit - tools, libraries and test resources for platformCI

**toolkit**, this project is for tools, libraries and test resources for platformCI. You may also find some code snippets here, hope it can help you too.

## fd - fake device framework

**fd** is a fake device framework, firstly designed in [npt](http://becrtt01.china.nsn-net.net/platformci/npt/). It is designed as a fake SSH service that can be simulated as mcRNC or Fews PC. We need it to build the unit testing for npt and coci-runner.

The architecture is as below:

        FakeDevice
            |
            |-start
                |-_run |-> (create new connection)
                       |   FakeServer
                       |   (create new thread) -> interact
                                                    |
                                                    |-> BaseCommandWrapper
                                                    |   (execute wrapper function)
                                                    |   (Return) |-> AbstractReturn |-> SuccessReturn -> SerializedSuccessReturn
                                                                 |                  |-> ErrorReturn
                                                                 |-> ExitReturn

### SSH Base Server

#### FakeDevice

It is the base class of fd. In this class, it will:

1. bind a TCP port, and listen as a TCP server
1. start a thread to accept connection of clients, and handle those connections

If you want to implement your own **fake** devices, you should inherit from this class. The open interfaces include:

| public function   | usage |
| ---------------   | ----- |
| get_port          | return the TCP port used by the FakeDevice, you should not override it |
| get_welcome       | return the welcome message of FakeDevice, override it if you need |
| get_exit          | return the exit command of FakeDevice, override it if you need |
| get_prompt        | return the prompt of connection of FakeDevice, override it if you need |
| register_commands | add extra commands handler to FakeDevice, it is used as an extension, but this feature has not been implemented yet |

#### FakeServer

This class is inheritted from paramiko.ServerInterface, the basic usage of this class is making fake device as a SSH server. The code may be like below:

```python
import select
import socket
import logging
import paramiko

TEST_RSA_KEY_PATH = os.path.join(os.path.dirname(__file__), 'test_rsa.key')
TEST_HOST_KEY = paramiko.RSAKey(filename=TEST_RSA_KEY_PATH)

sockl = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sockl.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sockl.bind(('', 0))
sockl.listen(100)

select.select([self.sockl], [], [], 1)
client, addr = sockl.accept()
logging.info('connected from %s:%s' % addr)
trans = paramiko.Transport(client)
trans.add_server_key(TEST_HOST_KEY)
server = FakeServer()
trans.start_server(server=server)

channel = trans.accept(20)
server.event.wait(10)

channel.send('welcome to Fake SSH Device')

f = channel.makefile('rU')
cmd = f.read()

channel.send(cmd)
channel.send('cmd response') # TODO: execute command here
```

### Command Wrapper

**BaseCommandWrapper** is the mapping of user command and functions. Each command defined in the subclass of BaseCommandWrapper can be used as a command of clients. For the function of BaseCommandWrapper, it has several kinds of return value:

1. ExitReturn   -   quit the client connection
1. AbstractReturn
    1. SuccessReturn    -   successful return, the error code is 0
        1. SerializedSuccessReturn  -   successful return, but the output is not in one transferring
    1. ErrorReturn  -   error return, the error code is not 0

For the AbstractReturn, if the `is_eof()` is true, the client connection will be closed.

An example of one simple command:

```python
def ls(self, *args):
    ''' an example to command function '''
    return SuccessReturn('base.py  base.pyc  bcn.py  fews.py')
```

#### Handle interactive command

For the interactive command, the command function should be a generator, and return an instance of AbstractReturn in the last loop. 

An example of interactive command:

```python
def reboot(self, *args):
    ''' an example to handle interactive command '''
    answer = (yield 'Are you sure (Y/N?): ')
    if answer == 'Y':
        yield  # NOTE: this line is necessary to get answer
        yield SuccessReturn('rebooting scheduled', True)
    else:
        yield
        yield ErrorReturn('cancelled!')
```

#### Handle serialized output of command

Sometimes we need to simulator a serialized return, the client cannot fetch the output in one time. We need to use SerializedSuccessReturn.

An example of serialized return command:

```python
def reset(self, *args):
    ''' an example to handle serialized command '''
    return SerializedSuccessReturn(['reset scheduled\n', 'reset successfully'])
```

#### Switch data between commands

BaseCommandWrapper has an attribute called `context` to handle data switching between different commands. 

An example of context usage:

```python
def echo(self, *args): 
    echo_str = ' '.join(args)
    if echo_str == '$?':
        yield SuccessReturn(str(self.context.get_error_code()))
    elif echo_str.count('\'') % 2 != 0:
        more_str = echo_str
        while True:
            more_str = (yield '')
            yield
            echo_str += '\n' + more_str
            if more_str.count('\'') % 2 != 0:
                echo_str = echo_str.strip("'")
                break
    if '>>' in echo_str or '>' in echo_str:
        _content, _file_name = re.split('>|>>', echo_str, 1)
        if hasattr(self.context, 'file_%s' % _file_name):
            file_content = getattr(self.context, 'file_%s' % _file_name)
            if '>>' in echo_str:
                setattr(self.context, 'file_%s' % _file_name, file_content + _content)
            else:
                setattr(self.context, 'file_%s' % _file_name, _content)
        yield SuccessReturn('')
    yield SuccessReturn(echo_str)
```

### Unit Test

### TODO: use gevent

Coding with thread is a risk. Using gevent can remove any thread related code. And the IO performance using gevent will be better than using thread, we should not need to use threadpool any more.

An echo server example of gevent:

```python
#!/usr/bin/env python
"""Simple server that listens on port 6000 and echos back every input to the client.

Connect to it with:
telnet localhost 6000

Terminate the connection by terminating telnet (typically Ctrl-] and then 'quit').
"""
from __future__ import print_function
from gevent.server import StreamServer


# this handler will be run for each incoming connection in a dedicated greenlet
def echo(socket, address):
    print('New connection from %s:%s' % address)
    socket.sendall('Welcome to the echo server! Type quit to exit.\r\n')
    # using a makefile because we want to use readline()
    fileobj = socket.makefile()
    while True:
        line = fileobj.readline()
        if not line:
            print("client disconnected")
            break
        if line.strip().lower() == 'quit':
            print("client quit")
            break
        fileobj.write(line)
        fileobj.flush()
        print("echoed %r" % line)


if __name__ == '__main__':
    # to make the server use SSL, pass certfile and keyfile arguments to the constructor
    server = StreamServer(('0.0.0.0', 6000), echo)
    # to start the server asynchronously, use its start() method;
    # we use blocking serve_forever() here because we have no other jobs
    print('Starting echo server on port 6000')
    server.serve_forever()
```

### TODO: support other protocols

Paramiko related code can be replaced or removed, so that FakeDevice become a pure framework, that should not be very complex.
