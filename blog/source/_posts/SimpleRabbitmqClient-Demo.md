---
title: SimpleRabbitmqClient Demo
date: 2018-11-15 19:36:56
tags: 
- SimpleRabbitmqClient
---

我就想把我写的关于SimpleRabbitmqClient的Demo上传一下，方便以后后人查阅吧

<!--more-->

**Consume:**

```c++
/*
 * @Author: Zerber(Lazy_Rabbit) 懒散兔 Zzzzz... 
 * @Date: 2018-11-14 15:11:01 
 * @Last Modified by: Zerber(Lazy_Rabbit) 懒散兔 Zzzzz...
 * @Last Modified time: 2018-11-14 16:45:26
 */

#include <mutex>
#include <thread>
#include <iostream>
#include <functional>
#include "SimpleAmqpClient/SimpleAmqpClient.h"

#define TEST_QUEUE_1 "test.Queue1"
#define TEST_QUEUE_2 "test.Queue2"

#define TEST_EXCHANGE_RABBIT "test.exchange.rabbit"
#define TEST_EXCHANGE_TIGER "test.exchange.tiger"

//Generate some methods for detecting interfaces

class ConsumeOpr {

typedef std::function<void (void)> TTestFun;
public:
    typedef std::function<void (std::string, std::string, std::string)> TQueueMsgHandler;

    struct Parm {
        std::string host; 
        int port;
        std::string userName;
        std::string password;
        
        Parm():port(0){}
    };  

    bool Init(const Parm& parm) {
        parm_ = parm;
        if(parm_.host.empty() || parm_.userName.empty() || parm_.password.empty() || parm_.port<=0 ) {
            std::cout << "init false!" << std::endl;
            return false ;
        }
        try {
            channel_ = AmqpClient::Channel::Create(parm_.host, parm_.port, parm_.userName, parm_.password);
        } catch (std::exception& e) {
            std::cout << "init false error[" << e.what() << std::endl;
            return false ;
        }

        return true;
    }

    //this demo is test for more exchanges bind in one queue.
    void TestTwoQueueAndExchanges(const std::string& queueName) {
        std::vector<std::string> queueNames;
        queueNames.push_back(queueName);

        TTestFun testFun = [&]() {
            //channel_->DeclareQueue(queueName, false, true, false, false);
            ListenLoop_(queueNames, ConsumeOpr::DefaultQueueMsgFun);
        };
        ThrowExceFrame_(testFun);
    }

    void TestMoreConsumeWithRoutingkey(const std::vector<std::string>& queueNames) {
        TTestFun testFun = [&]() {
            //for(std::string queueName : queueNames) {
                //channel_->DeclareQueue(queueName, false, true, false, false);
            //}
            ListenLoop_(queueNames, ConsumeOpr::DefaultQueueMsgFun);
        };
        
        ThrowExceFrame_(testFun);
    }

/////////////////////////////////////////

    static void DefaultQueueMsgFun(std::string msgExch, std::string msgBody, std::string consumeTag) {
        std::cout << "msgExch[" << msgExch << "] msgBody[" << msgBody << "] consumeTag[" << consumeTag << "]" << std::endl;
    }

private:
    void ThrowExceFrame_(TTestFun testFun) {
            try {
                testFun();
            } catch(std::exception& e) {
                std::cout << "test error[" << e.what()<< "]";
            }
    }

    void ListenLoop_(const std::vector<std::string>& queueNames, TQueueMsgHandler queueMsgHandler) {
        std::vector<std::string> consumerTags;
        
        for(std::string queueName : queueNames) {
            std::string consumeTag = channel_->BasicConsume(queueName);
            std::cout << "tag[" << consumeTag << "], queueName[" << queueName << "]" << std::endl;
            consumerTags.push_back(consumeTag);
        }

        while(true) {
            //BasicConsumeMessage有非vec版本，自己看接口了
            auto msg = channel_->BasicConsumeMessage(consumerTags);
            std::string exch = msg->Exchange();
            std::string body = msg->Message()->Body();
            std::string consumeTag = msg->ConsumerTag();
            queueMsgHandler(exch, body, consumeTag);
        }
    }

    Parm parm_;
    AmqpClient::Channel::ptr_t channel_;
};

/////////////////////////////////////////

int main() {

    ConsumeOpr::Parm consumeOprParm;
    consumeOprParm.host = "192.168.3.161";
    consumeOprParm.port = 5672;
    consumeOprParm.userName = "guest";
    consumeOprParm.password = "guest";

    ConsumeOpr consumeOpr;
    consumeOpr.Init(consumeOprParm);

    //consumeOpr.TestOneQueueAndExchanges(TEST_QUEUE_1);
    
    std::vector<std::string> queueNames;
    queueNames.push_back(TEST_QUEUE_1);
    queueNames.push_back(TEST_QUEUE_2);
    consumeOpr.TestMoreConsumeWithRoutingkey(queueNames);
}
```

**Producer:**

```c++
/*
 * @Author: Zerber(Lazy_Rabbit) 懒散兔 Zzzzz... 
 * @Date: 2018-11-14 15:10:55 
 * @Last Modified by: Zerber(Lazy_Rabbit) 懒散兔 Zzzzz...
 * @Last Modified time: 2018-11-14 16:48:27
 */

#include <iostream>
#include "SimpleAmqpClient/SimpleAmqpClient.h"

#define TEST_QUEUE_1 "test.Queue1"
#define TEST_QUEUE_2 "test.Queue2"

#define TEST_EXCHANGE_RABBIT "test.exchange.rabbit"
#define TEST_EXCHANGE_TIGER "test.exchange.tiger"

class ProducertOpr {

typedef std::function<void (void)> TTestFun;
public:
    struct Parm {
        std::string host; 
        int port;
        std::string userName;
        std::string password;
        
        Parm():port(0){}
    };  

    bool Init(const Parm& parm) {
        parm_ = parm;
        if(parm_.host.empty() || parm_.userName.empty() || parm_.password.empty() || parm_.port<=0 ) {
            std::cout << "init false!" << std::endl;
            return false ;
        }
        try {
            channel_ = AmqpClient::Channel::Create(parm_.host, parm_.port, parm_.userName, parm_.password);
        } catch (std::exception& e) {
            std::cout << "init false!" << std::endl;
            return false ;
        }

        return true;
    }

    //main thread loop
    void TestMoreExchageToOneQueueWithOutRoutingkey() {
        TTestFun testFun = [&]() {
            channel_->DeclareExchange(TEST_EXCHANGE_RABBIT, AmqpClient::Channel::EXCHANGE_TYPE_DIRECT);
            channel_->DeclareExchange(TEST_EXCHANGE_TIGER, AmqpClient::Channel::EXCHANGE_TYPE_DIRECT);
            channel_->DeclareQueue(TEST_QUEUE_1, false, true, false, false);
            channel_->BindQueue(TEST_QUEUE_1, TEST_EXCHANGE_RABBIT);
            channel_->BindQueue(TEST_QUEUE_1, TEST_EXCHANGE_TIGER);
            LoopToSendMsg_(false);
        };
        ThrowExceFrame_(testFun);
    }

    void TestMoreExchageToOneQueueWithRoutingkey() {
        TTestFun testFun = [&]() {
            channel_->DeclareExchange(TEST_EXCHANGE_RABBIT, AmqpClient::Channel::EXCHANGE_TYPE_DIRECT);
            channel_->DeclareExchange(TEST_EXCHANGE_TIGER, AmqpClient::Channel::EXCHANGE_TYPE_DIRECT);
            channel_->DeclareQueue(TEST_QUEUE_1, false, true, false, false);
            channel_->DeclareQueue(TEST_QUEUE_2, false, true, false, false);
            channel_->BindQueue(TEST_QUEUE_1, TEST_EXCHANGE_RABBIT, "rabbit");
            channel_->BindQueue(TEST_QUEUE_2, TEST_EXCHANGE_TIGER, "tiger");
            LoopToSendMsg_(true);
        };
        ThrowExceFrame_(testFun);
    }

/////////////////////////////////////////

private:
    void ThrowExceFrame_(TTestFun testFun) {
        try {
            testFun();
        } catch(std::exception& e) {
            std::cout << "test error[" << e.what()<< "]";
        }
    }

    void LoopToSendMsg_(bool isRouteKey) {
        std::string exchangeName;
        std::string queueName;
        std::string message;
        std::string routeKey;

        if(!isRouteKey) {
            while(true) {
                std::cout << "write [exchange] [queueName] [message] to send:" << std::endl;
                std::cin >> exchangeName >> queueName >> message;

                channel_->BasicPublish(exchangeName, "", AmqpClient::BasicMessage::Create(message));
                std::cout << "exchangeName[" << exchangeName << "] , queueName[" << queueName << "] , message[" << message << "] has been send." << std::endl;
            }
        }
        else {
            while(true) {
                std::cout << "write [exchange] [routeKey] [message] to send:" << std::endl;
                std::cin >> exchangeName  >> routeKey >> message;

                channel_->BasicPublish(exchangeName, routeKey, AmqpClient::BasicMessage::Create(message));
                std::cout << "exchangeName[" << exchangeName << "] , routeKey[" << routeKey << 
                             "] , message[" << message << "] has been send." << std::endl;
            }
        }
    }

    Parm parm_;
    AmqpClient::Channel::ptr_t channel_;
};

/////////////////////////////////////////

int main() {

    ProducertOpr::Parm producertOprParm;
    producertOprParm.host = "192.168.3.161";
    producertOprParm.port = 5672;
    producertOprParm.userName = "guest";
    producertOprParm.password = "guest";

    ProducertOpr producertOpr;
    producertOpr.Init(producertOprParm);

    //match consume fun [TestTwoQueueAndExchanges]
    //producertOpr.TestMoreExchageToOneQueueWithOutRoutingey();

    //match consume fun [TestMoreThreadWithRoutingkey]
    producertOpr.TestMoreExchageToOneQueueWithRoutingkey();
    return 0;
}

```