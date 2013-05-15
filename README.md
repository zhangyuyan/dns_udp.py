from twisted.names.client import AXFRController
from twisted.names.dns import DNSDatagramProtocol
from twisted.names.dns import  Query
from twisted.names.dns import Name
from twisted.names.dns import TXT
from twisted.names.dns import CH
from twisted.names.dns import Message
from twisted.python.failure import Failure
from twisted.names.error import DNSQueryTimeoutError

import struct
import random
import warnings
import types
import socket

from twisted.internet import defer
from twisted.internet.defer import Deferred
from twisted.internet import reactor
from twisted.internet import task
from twisted.internet import reactor
reactor.suggestThreadPoolSize(50000)

write_error = open('err','a')
write_another = open('another_err','a')
write_result = open('success','a')

class NullAnswer(Exception): 
    pass
class NullQuery(Exception): 
    pass
class MalResponse(Exception): 
    pass

class DnsDatagramProtocol(DNSDatagramProtocol):
    def datagramReceived(self, data, addr):
        m = Message()
        try:
            m.fromStr(data)
        except EOFError:
            log.msg("Truncated packet (%d bytes) from %s" % (len(data), addr))
            return
        except:
            log.err(failure.Failure(), "Unexpected decoding error")
            return
        if m.id in self.liveMessages:
            d, canceller = self.liveMessages[m.id]
            del self.liveMessages[m.id]
            canceller.cancel()
            try:
                d.callback(m)
            except:
                log.err()
        else:
            if m.id not in self.resends:
                self.controller.messageReceived(m, self)

def getResult(result,ip):
    if result.answer == 0: 
        raise MalResponse('MalResponse:Set Q flag,not A')
    if len(result.queries)!= 0:
        queries = result.queries.pop()
    else:
        raise NullQuery('NullQuery: not implemented')
    if len(result.answers)!= 0:
        res = ''
        try:
            res = result.answers[0].payload.data[0]
        except Exception, e:
            pass
        write_result.write(ip+'\t'+res+'\n')
        write_result.flush()
    else:
        raise NullAnswer('NullAnswer: refused')

def getError(reason,ip):   
    msg = '%s\t'%(ip)
    if reason.check(DNSQueryTimeoutError):
        write_error.write(msg + 'Timed out' + '\n')
        write_error.flush()
    elif reason.check(NullAnswer):
        write_error.write(msg + 'Set Null Answer:refused' + '\n')
        write_error.flush()
    elif reason.check(NullQuery):
        write_error.write(msg + 'Set Null Query:not implemented' + '\n')
        write_error.flush()
    elif reason.check(MalResponse):
        write_error.write(msg + 'Set Q Flag,not A: secured?' + '\n')
        write_error.flush()
    else:
        # write_error.write(msg + 'other err' + '\n')
        # write_error.flush()
        rea = reason.getErrorMessage()
        write_another.write(msg+'\t'+rea+'\n')
        write_another.flush()
        # print msg + rea

def doWork():
    i = 1
    fp = file("dns.log", 'w')
    for ip in file("list12.txt"):
    # for ip in file("list12"):
        ip = ip.strip()
        print "query %d\t%s......"%(i,ip)
        d = Deferred()
        name = Name('version.bind')
        axf = AXFRController(name,d)
        dns = DnsDatagramProtocol(axf)
        d1 = dns.query((ip,53), [Query('version.bind',TXT,CH)])
        d1.addCallback(getResult,ip)
        d1.addErrback(getError,ip)
        logRecord = "%d\t%s\n" %(i,ip)
        i += 1
        fp.write(logRecord)
        fp.flush()
        yield d1

def finish(igo):
    reactor.callLater(5, reactor.stop)
    write_error.close()
    write_result.close()
    write_another.close()

def taskRun():
    deferreds = []
    coop = task.Cooperator()
    work = doWork()
    maxRun = 2000
    for i in xrange(maxRun):
        d = coop.coiterate(work)
        deferreds.append(d)
    dl = defer.DeferredList(deferreds, consumeErrors=True)
    dl.addCallback(finish)

def main():
    taskRun()
    reactor.run()

if __name__ == '__main__':
    main()
