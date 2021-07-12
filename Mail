#include "player_mail.h"

#include "gs/netmsg/send_to_master.h"
#include "gs/global/dbgprt.h"
#include "gs/global/msg_pack_def.h"
#include "gs/player/subsys_if.h"
#include "gs/player/player_sender.h"
#include "gs/item/item_data.h"
#include "gs/item/item.h"
#include "gs/item/item_manager.h"

#include "common/obj_data/player_attr_util.h"
#include "common/protocol/gen/G2M/mail_msg.pb.h"


namespace gamed
{
	
using namespace std;
using namespace shared::net;
using namespace common::protocol;

const int32_t MAILBOX_MAX_SIZE = 256;

PlayerMail::PlayerMail(Player& player)
	: PlayerSubSystem(SUB_SYS_TYPE_MAIL, player)
{
	SAVE_LOAD_REGISTER(common::PlayerMailList, PlayerMail::SaveToDB, PlayerMail::LoadFromDB);
}

PlayerMail::~PlayerMail()
{
}

bool PlayerMail::LoadFromDB(const common::PlayerMailList& data)
{
	for (size_t i = 0; i < data.mail_list.size(); ++i)
	{
		const Mail& mail = data.mail_list[i];
		mail_list_[mail.id] = mail;
	}
	return true;
}

bool PlayerMail::SaveToDB(common::PlayerMailList* pData)
{
	pData->mail_list.clear();
	MailList::const_iterator it = mail_list_.begin();
	for (; it != mail_list_.end(); ++it)
	{
		pData->mail_list.push_back(it->second);
	}
	return true;
}

void PlayerMail::RegisterCmdHandler()
{
	REGISTER_NORMAL_CMD_HANDLER(C2G::GetMailList, PlayerMail::CMDHandler_GetMailList);
	REGISTER_NORMAL_CMD_HANDLER(C2G::GetMailAttach, PlayerMail::CMDHandler_GetMailAttach);
	REGISTER_NORMAL_CMD_HANDLER(C2G::DeleteMail, PlayerMail::CMDHandler_DeleteMail);
	REGISTER_NORMAL_CMD_HANDLER(C2G::TakeoffMailAttach, PlayerMail::CMDHandler_TakeoffMailAttach);
	REGISTER_NORMAL_CMD_HANDLER(C2G::SendMail, PlayerMail::CMDHandler_SendMail);
}

void PlayerMail::RegisterMsgHandler()
{
	REGISTER_MSG_HANDLER(GS_MSG_GET_ATTACH_REPLY, PlayerMail::MSGHandler_GetAttachRe);
	//REGISTER_MSG_HANDLER(GS_MSG_DELETE_MAIL_REPLY, PlayerMail::MSGHandler_DeleteMailRe);
	REGISTER_MSG_HANDLER(GS_MSG_SEND_MAIL_REPLY, PlayerMail::MSGHandler_SendMailRe);
	REGISTER_MSG_HANDLER(GS_MSG_ANNOUNCE_NEW_MAIL, PlayerMail::MSGHandler_NewMail);
}

void PlayerMail::AnnounceNewMail()
{
	MailList::const_iterator mit = mail_list_.begin();
	for (; mit != mail_list_.end(); ++mit)
	{
		if (readed(mit->second.attr) || need_delete(mit->second.op))
		{
			continue;
		}

		__PRINTF("AnnounceNewMail roleid=%ld", player_.role_id());
		G2C::AnnounceNewMail packet;
		player_.sender()->SendCmd(packet);
		return;
	}
}

static void Convert(G2C::Mail& lmail, const Mail& rmail)
{
	lmail.id = rmail.id;
	lmail.attr = rmail.attr;
	lmail.time = rmail.time;
	lmail.sender = rmail.sender;
	lmail.sender_name = rmail.name;
	lmail.title = rmail.title;
	lmail.content = rmail.content;
}

void PlayerMail::CMDHandler_GetMailList(const C2G::GetMailList& packet)
{
	G2C::GetMailList_Re reply;
	MailList::const_iterator it = mail_list_.begin();
	for (; it != mail_list_.end(); ++it)
	{
		if (it->first <= packet.max_mailid || need_delete(it->second.op))
		{
			continue;
		}

		G2C::Mail mail;
		Convert(mail, it->second);
		reply.mail_list.push_back(mail);
	}

	int32_t mail_size = (int32_t)reply.mail_list.size();
	__PRINTF("GetMailList roleid=%ld mail.size=%d", player_.role_id(), mail_size);
	player_.sender()->SendCmd(reply);
}

void PlayerMail::CMDHandler_GetMailAttach(const C2G::GetMailAttach& packet)
{
	int64_t mail_id = packet.mail_id;
	__PRINTF("GetMailAttach roleid=%ld mailid=%ld", player_.role_id(), mail_id);
	MailList::iterator mit = mail_list_.find(mail_id);
	if (mit == mail_list_.end() || need_delete(mit->second.op))
	{
		return;
	}
	if (!has_attach(mit->second.attr))
	{
		set_readed(mit->second.op, mit->second.attr);
		return;
	}

	AttachList::const_iterator ait = attach_list_.find(mail_id);
	if (ait != attach_list_.end())
	{
		G2C::GetMailAttach_Re reply;
		reply.attach = ait->second;
		player_.sender()->SendCmd(reply);
		set_readed(mit->second.op, mit->second.attr);
	}
	else
	{
		G2M::GetMailAttach proto;
		proto.set_roleid(player_.role_id());
		proto.set_mailid(mail_id);
		player_.sender()->SendToMaster(proto);
	}
}

void PlayerMail::CMDHandler_DeleteMail(const C2G::DeleteMail& packet)
{
	int64_t mail_id = packet.mail_id;
	__PRINTF("DeleteMail roleid=%ld mailid=%ld", player_.role_id(), mail_id);
	MailList::iterator mit = mail_list_.find(mail_id);
	if (mit == mail_list_.end() || need_delete(mit->second.op))
	{
		return;
	}

	set_delete(mit->second.op);
	G2C::DeleteMail_Re reply;
	reply.mail_id = mail_id;
	player_.sender()->SendCmd(reply);
	//G2M::DeleteMail proto;
	//proto.set_roleid(player_.role_id());
	//proto.set_mailid(mail_id);
	//proto.set_attach(has_attach(mit->second.attr));
	//player_.sender()->SendToMaster(proto);
}

void PlayerMail::CMDHandler_TakeoffMailAttach(const C2G::TakeoffMailAttach& packet)
{
	int64_t mail_id = packet.mail_id;
	__PRINTF("CMDHandler_TakeoffMailAttach mail_id=%ld", mail_id);
	MailList::iterator mit = mail_list_.find(mail_id);
	if (mit == mail_list_.end() || !has_attach(mit->second.attr) || need_delete(mit->second.op))
	{
		return;
	}

	AttachList::iterator ait = attach_list_.find(mail_id);
    if (ait == attach_list_.end())
    {
	    __PRINTF("CMDHandler_TakeoffMailAttach mail_id=%ld Attach Not Found", mail_id);
        return;
    }

	int32_t where = Item::INVENTORY;
	G2C::MailAttach& attach = ait->second;
	size_t item_size = attach.attach_item.item_list.size();
	if (!player_.HasSlot(where, item_size))
	{
		player_.sender()->ErrorMessage(G2C::ERR_INV_FULL);
		return;
	}

    player_.AddCash(attach.attach_cash);
    if (attach.attach_cash > 0)
    {
	    __PRINTF("CMDHandler_TakeoffMailAttach mail_id=%ld cash=%d", mail_id, attach.attach_cash);
    }
	player_.GainScore(attach.attach_score);
	for (size_t i = 0; i < item_size; ++i)
	{
		const G2C::ItemDetail& detail = attach.attach_item.item_list[i];

		itemdata item;
		MakeItemdata(item, detail);
		player_.GainItem(where, item, GIM_MAIL);
	}

	G2C::TakeoffMailAttach_Re reply;
	reply.result = 1;
	reply.mail_id = mail_id;
	player_.sender()->SendCmd(reply);
	takeoff_attach(mit->second.op, mit->second.attr);

	// 强制玩家存盘，保证邮件附件与包裹一致
	player_.ResetWriteTime();
}

#define MAX_ATTACH_NUM 5
#define MAIL_TAX 0.05

static bool CheckSendMailParam(const C2G::SendMail& packet)
{
	// receiver必须要为有效的roleid
	//除钱跟经验外，最多可以发送4个附件
	if (packet.receiver == 0 || packet.attach_item.size() > MAX_ATTACH_NUM)
	{
		return false;
	}
	return true;
}

static bool HasAttachItem(Player& player, const C2G::SendMail::ItemInfoVec& item_list)
{
	for (size_t i = 0; i < item_list.size(); ++i)
	{
		const C2G::SendMail::ItemInfo& info = item_list[i];
		if (!player.CheckItem(info.index, info.type, info.count))
		{
			return false;
		}
	}
	return true;
}

static uint32_t CalcPostage(Player& player, const C2G::SendMail& packet)
{
	uint32_t postage = 0;
	for (size_t i = 0; i < packet.attach_item.size(); ++i)
	{
		const C2G::SendMail::ItemInfo& info = packet.attach_item[i];
		itemdata data;
		player.QueryItem(Item::INVENTORY, info.index, data);
		postage += info.count * data.recycle_price;
	}
	postage *= MAIL_TAX;
	return postage + 1;
}

static bool TakeoutItem(Player& player, const C2G::SendMail::ItemInfoVec& item_list, string& str_attach)
{
	G2C::ItemDetailVec tmp;
	for (size_t i = 0; i < item_list.size(); ++i)
	{
		const C2G::SendMail::ItemInfo& info = item_list[i];
		itemdata data;
		player.QueryItem(Item::INVENTORY, info.index, data);
		data.count = info.count;
		G2C::ItemDetail detail;
		MakeItemDetail(detail, data);
		tmp.item_list.push_back(detail);
		if (!player.TakeOutItem(info.index, info.type, info.count))
		{
			return false;
		}
	}
	if (!tmp.item_list.empty())
	{
		ByteBuffer buffer;
		tmp.Pack(buffer);
		str_attach.append((const char*)buffer.contents(), buffer.size());
	}
	return true;
}

void PlayerMail::CMDHandler_SendMail(const C2G::SendMail& packet)
{
	ASSERT(player_.role_id() == packet.sender);
	__PRINTF("CMDHandler_SendMail role_id=%ld", packet.sender);
	if (!CheckSendMailParam(packet))
	{
		return;
	}
	// 检查是否有附件物品
	if (!HasAttachItem(player_, packet.attach_item))
	{
		return;
	}
    // 检查是否有足够学分
    if (!player_.CheckScore(packet.attach_score))
    {
        return;
    }
	// 计算邮资
	uint32_t postage = CalcPostage(player_, packet);
	if (!player_.CheckMoney(postage))
	{
		return;
	}
	// 扣除附件物品
	string str_attach;
	if (!TakeoutItem(player_, packet.attach_item, str_attach))
	{
		return;
	}
	player_.SpendScore(packet.attach_score);
	player_.SpendMoney(postage);

	G2M::SendMail proto;
	proto.set_sender(packet.sender);
	proto.set_receiver(packet.receiver);
	proto.set_attach_score(packet.attach_score);
	proto.set_name(packet.name);
	proto.set_title(packet.title);
	proto.set_content(packet.content);
	proto.set_attach_item(str_attach);
	player_.sender()->SendToMaster(proto);

	// 强制玩家存盘，保证邮件附件与包裹一致
	// 如果邮件发送失败通过系统退信来进行补偿
	player_.ResetWriteTime();
}

void PlayerMail::SendSysMail(int32_t score, int32_t item_id)
{
    playerdef::SysMail sysmail;
    sysmail.attach_score  = score;
    sysmail.sender       = "GM";
    sysmail.title        = "test sys mail";
    sysmail.content      = "hello world";
    playerdef::MailAttach attach;
    attach.id            = item_id;
    attach.count         = 1;
    sysmail.attach_list.push_back(attach);
    shared::net::ByteBuffer buf;
    sysmail.Pack(buf);
    SendSysMail(buf);
}

void PlayerMail::SendSysMail(ByteBuffer& data)
{
    SendSysMail(player_.master_id(), player_.role_id(), data);
}

void PlayerMail::SendSysMail(int32_t master_id, RoleID receiver, shared::net::ByteBuffer& data)
{
    playerdef::SysMail sysmail;
	sysmail.UnPack(data);

    SendSysMail(master_id, receiver, sysmail);
}

void PlayerMail::SendSysMail(int32_t master_id, RoleID receiver, const playerdef::SysMail& sysmail)
{
	G2C::ItemDetailVec item;
	for (size_t i = 0; i < sysmail.attach_list.size(); ++i)
	{
		const playerdef::MailAttach& attach = sysmail.attach_list[i];		
	    itemdata item_data;
		if (!s_pItemMan->GenerateItem(attach.id, item_data))
            continue;

		item_data.count = attach.count;

		G2C::ItemDetail detail;
		MakeItemDetail(detail, item_data);
		item.item_list.push_back(detail);
	}
	ByteBuffer buffer;
	item.Pack(buffer);
	string str_attach = "";
	str_attach.append((const char*)buffer.contents(), buffer.size());

	G2M::SendMail proto;
	proto.set_sender(0);
	proto.set_receiver(receiver);
	proto.set_attach_score(sysmail.attach_score);
	proto.set_name(sysmail.sender);
	proto.set_title(sysmail.title);
	proto.set_content(sysmail.content);
	proto.set_attach_item(str_attach);
	NetToMaster::SendProtocol(master_id, proto);
}



// MsgHandler
int32_t PlayerMail::MSGHandler_GetAttachRe(const MSG& msg)
{
	msgpack_get_attach_reply param;
	MsgContentUnmarshal(msg, param);

	__PRINTF("GetAttachRe roleid=%ld mailid=%ld", player_.role_id(), param.mailid);
	G2C::MailAttach& attach = attach_list_[param.mailid];
	attach.id = param.mailid;
	attach.attach_cash = param.attach_cash;
	attach.attach_score = param.attach_score;
	if (!param.attach_item.empty())
	{
		ByteBuffer buffer;
		buffer.append(param.attach_item);
		attach.attach_item.UnPack(buffer);
	}

	G2C::GetMailAttach_Re reply;
	reply.attach = attach;
	player_.sender()->SendCmd(reply);

	MailList::iterator it = mail_list_.find(param.mailid);
	ASSERT(it != mail_list_.end());
	set_readed(it->second.op, it->second.attr);
	return 0;
}
/*
int32_t PlayerMail::MSGHandler_DeleteMailRe(const MSG& msg)
{
	CHECK_CONTENT_PARAM(msg, msg_delete_mail_reply);
	msg_delete_mail_reply& param = *(msg_delete_mail_reply*)msg.content;

	mail_list_.erase(param.mailid);
	attach_list_.erase(param.mailid);

	G2C::DeleteMail_Re reply;
	reply.mail_id = param.mailid;
	player_.sender()->SendCmd(reply);
	return 0;
}
*/
int32_t PlayerMail::MSGHandler_SendMailRe(const MSG& msg)
{
	CHECK_CONTENT_PARAM(msg, msg_send_mail_reply);
	msg_send_mail_reply& param = *(msg_send_mail_reply*)msg.content;
	G2C::SendMail_Re reply;
	reply.mail_id = param.mailid;
	player_.sender()->SendCmd(reply);
	return 0;
}

int32_t PlayerMail::MSGHandler_NewMail(const MSG& msg)
{
	msgpack_announce_new_mail param;
	MsgContentUnmarshal(msg, param);

	Mail& mail = mail_list_[param.mailid];
	mail.op = 0;
	mail.id = param.mailid;
	mail.time = param.time;
	mail.sender = param.sender;
	mail.name = param.name;
	mail.title = param.title;
	mail.content = param.content;
	mail.attr = 0;
	if (param.attach_cash != 0 || param.attach_score != 0 || !param.attach_item.empty())
	{
		set_attach(mail.attr);
	}
	if (mail.sender != 0)
	{
		set_player_mail(mail.attr);
	}

	G2C::MailAttach& attach = attach_list_[mail.id];
	attach.id = mail.id;
	attach.attach_cash = param.attach_cash;
	attach.attach_score = param.attach_score;
	if (!param.attach_item.empty())
	{
		ByteBuffer buffer;
		buffer.append(param.attach_item);
		attach.attach_item.UnPack(buffer);
	}

	G2C::AnnounceNewMail packet;
	player_.sender()->SendCmd(packet);
	return 0;
}

} // namespace gamed
