# walga_backend

![Untitled (1)](https://github.com/user-attachments/assets/fcd5bf94-0375-4984-ae32-83f08f43507e)




- **R&R : 백엔드 개발 (3명 中 1명)**
- **사용 기술 :** TypeScript, Express.js, GraphQL, PostgreSQL
- **소속 :** Startup-Express, 일진창업지원센터, 2024 청업중심대학학

🔗 **링크**

**홈페이지 링크** : https://walga.co.kr/

**소개 영상** : https://youtu.be/nUoKxUJZCco?si=7WyQDC2hmfgvD2v5

**구글 플레이 스토어** : https://play.google.com/store/apps/details?id=com.walcome.walgaAdminApp&hl=ko&gl=US

**앱스토어** : https://www.apple.com/kr/search/Walga?src=serp

**Github** : 팀 레포지토리에 있어 공개가 불가능합니다. 양해 부탁드립니다.



## 개발 기여 파트


### **1. 강아지 이용권 step, 예약 status**

- 강아지 이용권 step (대기,승인,거절,정지,취소 등) 변경 및 변경에 따른 예약 상태 변경

### 2. 이용권 예약단계별 FCM 알림 메세지 및 채팅 메세지 발신

- 유저 ↔ 원장의 상호간 알림을 FCM 토큰을 활용해 유저를 구분하여 메세지 발신

### 3. AWS 설정 및 리전 변경

### 4 . 비밀번호 암호화

- bcryptjs 라이브러리를 사용하여 비밀번호 암호화

## 성장 경험


### 1. 코드의 재사용성

- 재사용이 가능한 코드를 위해 각 로직에서 받을 argument, type 을 정의하고 코드 리팩토링을 습관화했습니다.

### 2. 가독성 높은 코드를 위한 노력

- 제가 구현한 코드의 일부입니다  (토글 클릭)
    
    ```tsx
    export const updateDogTicket = mutationField('updateDogTicket', {
      type: DogTicketType,
      args: {
        where: nonNull(DogTicketUpdateWhereInput),
        update: DogTicketUpdateInput,
      },
      resolve: async (_, { where, update }: any, ctx: Context) => {
        const { visible = true } = update
        if (!visible) {
          updateReservationVisibleToFalse(where.id, ctx)
          return ctx.prisma.DogTicket.update({ where, data: update })
        }
        if (!!update.step) {
          return await updateDogTicketStep(where.id, update.step, ctx)
        }
        return await updateDogTicketInfo(where.id, update, ctx)
      },
    })
    ```
    
    ```tsx
    export const updateDogTicketStep = async (
      dogTicketId: number,
      updateStep: number,
      ctx: Context,
    ) => {
      switch (updateStep) {
        case DOGTICKET_STEP.APPROVE:
        case DOGTICKET_STEP.REJECT:
        case DOGTICKET_STEP.STOPPED:
          return updateByWorker(dogTicketId, updateStep, ctx)
        case DOGTICKET_STEP.CANCEL:
          return updateByFamily(dogTicketId, updateStep, ctx)
        default:
      }
    }
    ```
    
    ```tsx
    const updateByWorker = async (
      dogTicketId: number,
      updateStep: number,
      ctx: Context,
    ) => {
      const dogTicket = await updateDogTicket(
        dogTicketId,
        { step: updateStep },
        ctx,
      )
      const sendData = await createDogTicketSendData(dogTicket, updateStep, ctx)
      switch (updateStep) {
        case DOGTICKET_STEP.APPROVE: {
          addNewCustomer(dogTicket.Dog.id, dogTicket.Shop.id, ctx)
          updateReservationAttendToApproval(dogTicket.id, ctx)
          if (dogTicket.User.notificationSetting[2]) {
            sendDogTicketNotification(sendData)
          }
        }
        case DOGTICKET_STEP.REJECT: {
          updateReservationVisibleToFalse(dogTicket.id, ctx)
          if (dogTicket.User?.notificationSetting[3]) {
            sendDogTicketNotification(sendData)
          }
        }
        case DOGTICKET_STEP.STOPPED: {
          updateApprovedReservationVisibleToFalse(dogTicket.id, ctx)
          sendDogTicketNotification(sendData)
        }
      }
      return dogTicket
    ```
    
    ```tsx
    export const sendDogTicketNotification = async ({
      senderAndReceiver,
      notiData,
      dogTicket,
      chatMessageData,
      ctx,
    }: {
      senderAndReceiver: SenderAndReceiver
      notiData: NotificationData
      dogTicket: DogTicketChatMessageData
      chatMessageData: ChatMessageData
      ctx: Context
    }) => {
      const chatRoom = await upsertChatRoomByDogIdAndShopId(
        chatMessageData.dogId,
        chatMessageData.shopId,
        ctx,
      )
      if (notiData.updateStep === DOGTICKET_STEP.PENDING) {
        const userIds = [senderAndReceiver.sender, senderAndReceiver.receiver]
        for (const userId of userIds) {
          await upsertChatRoomUserByUserIdAndChatRoomId(userId, chatRoom.id, ctx)
        }
      }
      const chatMessageDataWhere: SendChatMessageByTypeWhere = {
        chatRoomId: chatRoom.id,
        senderId: senderAndReceiver.sender,
      }
      const chatMessageDataCreate: SendChatMessageByTypeCreate =
        createDogTicketChatMessageBody(notiData.updateStep, notiData.senderName)
      const { chatRoomUsers } = await sendChatMessageByType(
        chatMessageDataWhere,
        chatMessageDataCreate,
        dogTicket,
        notiData.updateStep,
        ctx,
      )
      switch (notiData.updateStep) {
        case DOGTICKET_STEP.PENDING: {
          const notificationMessage: NotificationMessage = {
            notificationTitle: `${notiData.dogName}(${notiData.senderName}) 예약 도착`,
            notificationBody: `채팅방에서 ${notiData.ticketName} 예약을 확인해주세요.`,
            fcmMessage: {
              title: `WalgaAdmin`,
              body: `${notiData.dogName}(${notiData.senderName})님이 ${notiData.ticketName}을 예약했습니다.`,
              type: NotificationKinds.chat,
              data: {
                dogId: chatMessageData.dogId,
                shopId: chatMessageData.shopId,
                chatRoomId: chatRoom.id,
              },
            },
          }
          await sendNotification(senderAndReceiver, notificationMessage, ctx)
          break
        }
        case DOGTICKET_STEP.APPROVE: {
          const notificationMessage: NotificationMessage = {
            notificationTitle: `[${notiData.senderName}] ${notiData.ticketName} 승인`,
            notificationBody: `${notiData.dogName} 채팅방에서 메시지를 확인해주세요.`,
            fcmMessage: {
              title: `Walga`,
              body: `${notiData.senderName}에서 ${notiData.ticketName} 예약 승인되었습니다.`,
              type: NotificationKinds.chat,
              data: {
                dogId: chatMessageData.dogId,
                shopId: chatMessageData.shopId,
                chatRoomId: chatRoom.id,
              },
            },
          }
          await sendNotification(senderAndReceiver, notificationMessage, ctx)
          break
        }
        case DOGTICKET_STEP.REJECT: {
          const notificationMessage: NotificationMessage = {
            notificationTitle: `[${notiData.senderName}] ${notiData.ticketName} 승인 거부`,
            notificationBody: `${notiData.dogName} 채팅방에서 메시지를 확인해주세요.`,
            fcmMessage: {
              title: `Walga`,
              body: `${notiData.senderName}에서 ${notiData.ticketName} 예약 거부되었습니다.`,
              type: NotificationKinds.chat,
              data: {
                dogId: chatMessageData.dogId,
                shopId: chatMessageData.shopId,
                chatRoomId: chatRoom.id,
              },
            },
          }
          await sendNotification(senderAndReceiver, notificationMessage, ctx)
          break
        }
        case DOGTICKET_STEP.STOPPED: {
          break
        }
        case DOGTICKET_STEP.CANCEL: {
          const notificationMessage: NotificationMessage = {
            notificationTitle: `${notiData.dogName}(${notiData.senderName}) 예약 취소`,
            notificationBody: `${notiData.ticketName} 예약이 취소되었어요.`,
            fcmMessage: {
              title: `WalgaAdmin`,
              body: `${notiData.dogName}(${notiData.senderName})님이 ${notiData.ticketName} 예약 취소했습니다.`,
              type: NotificationKinds.chat,
              data: {
                dogId: chatMessageData.dogId,
                shopId: chatMessageData.shopId,
                chatRoomId: chatRoom.id,
              },
            },
          }
          await sendNotification(senderAndReceiver, notificationMessage, ctx)
          break
        }
        case DOGTICKET_CHATMESSAGE_STEP.DELIVE:
        case DOGTICKET_CHATMESSAGE_STEP.EDITTED: {
          const sendPromises = chatRoomUsers
            .filter(
              (chatRoomUser: { userId: string }) =>
                chatRoomUser.userId !== senderAndReceiver.sender,
            )
            .flatMap((chatRoomUser: { User: { fcmToken: string[] } }) =>
              chatRoomUser.User.fcmToken.map((token: string) => {
                const message = buildMessage({
                  notificationMessage: {
                    title: 'Walga',
                    body: chatMessageDataCreate.message,
                    type: NotificationKinds.chat,
                    data: {
                      chatRoomId: chatRoom.id,
                      dogId: chatMessageData.dogId,
                      shopId: chatMessageData.shopId,
                    },
                  },
                  fcmToken: token,
                })
                return sendFcmMessage(message)
              }),
            )
          await Promise.all(sendPromises)
          break
        }
        default:
          throw new GraphQLError(
            'Invalid step. Step must be 0 ~ 7 or not null or not undefined.',
          )
      }
    }
    
    ```
    

typescript 문서의 컨벤션을 지킨 코드 작성, enum 활용, 로직 함수화, switch-case문의 활용 등으로 가독성 높은 코드를 작성하는데 노력했습니다.

### 3. 팀원 상호간 코드 리뷰

코드의 로직 미스, 오타, 가독성 등을 상호 체크해주며 직접 코드를 작성하는 것 뿐만 아닌 코드를 리뷰하는 것이 개발 능력 향상에 도움이 되는 것을 체감했습니다.

### 4. 협업 도구의 적극 활용 (Jira, Git)

협업 툴 Git, Jira 등을 사용하며 이슈를 기반으로 브랜치를 관리하고 커밋에 대한 컨밴션을 지키며 개발자들간의 원할한 협업이 가능하도록 노력했습니다.
