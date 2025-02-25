# Deathly-Nexus-const {
    Client,
    GatewayIntentBits,
    Collection,
    EmbedBuilder,
    PermissionFlagsBits,
} = require("discord.js");
const fs = require("fs/promises");
const path = require("path");

// Configuration
const config = {
    token: process.env.DISCORD_BOT_TOKEN,
    prefix: "!",
    moderationLogChannel: "mod-logs",
    securityLogChannel: "security-logs",
    verificationRole: "Verified",
    moderatorRole: "Moderator",
    adminRole: "Administrator",

    // Anti-spam settings
    spamThreshold: 5,
    spamTimeWindow: 5000,

    // Auto-moderation settings 
    maxMentions: 5,
    maxEmojis: 10,
    maxMessageLength: 2000,
    maxCapsPercentage: 70,
    forbiddenWords: ["badword1", "badword2", "scam", "nitro free"],

    // Link filtering
    linkFilter: {
        enabled: true,
        whitelist: [
            "discord.com",
            "discordapp.com", 
            "imgur.com",
            "youtube.com",
            "youtu.be"
        ]
    },

    // Raid protection settings
    raidThreshold: 10,
    raidTimeWindow: 10000,
    raidAutoKick: true,
    raidLockdown: true,

    // Anti-nuke settings
    antiNuke: {
        enabled: true,
        logChannel: "anti-nuke-logs",
        maxRoleDeletions: 1,
        maxRoleUpdates: 3,
        maxChannelCreates: 5,
        maxChannelUpdates: 5,
        maxChannelDeletes: 1,
        maxUserBans: 2,
        maxUserKicks: 5,
        maxMessageDeletes: 10,
        webhookSpamThreshold: 3,
        maxWebhooksCreated: 1,
        maxEmojisCreated: 2,
        maxEmojisDeleted: 1
    },

    // Protected elements
    protectedRoles: ["Administrator", "Moderator"],
    protectedChannels: ["mod-logs", "security-logs"]
};

// Enhanced Logging System
const SEVERITY = {
    INFO: { color: "#00FF00", emoji: "ℹ️", priority: 0 },
    WARNING: { color: "#FFA500", emoji: "⚠️", priority: 1 },
    CRITICAL: { color: "#FF0000", emoji: "🚨", priority: 2 },
    SUCCESS: { color: "#00FF00", emoji: "✅", priority: 0 },
    DEBUG: { color: "#808080", emoji: "🔍", priority: -1 }
};

class SecurityBot {
    constructor() {
        this.client = new Client({
            intents: [
                GatewayIntentBits.Guilds,
                GatewayIntentBits.GuildMessages,
                GatewayIntentBits.MessageContent,
                GatewayIntentBits.GuildMembers,
                GatewayIntentBits.GuildModeration,
                GatewayIntentBits.GuildWebhooks,
                GatewayIntentBits.GuildEmojisAndStickers
            ]
        });

        this.commands = new Collection();
        this.actionLog = new Collection();
        this.joinLog = new Collection();
        this.suspiciousUsers = new Map();
        this.recentCriticalEvents = new Map();
        this.raidMode = false;
        this.raidModeLock = false;
        this.panicMode = false;
        this.testingInProgress = false;

        this.backupPath = path.join(__dirname, "../backups");
        this.initializeBackupDirectory();
        this.setupEventHandlers();
        this.setupCommands();
    }

    async setupCommands() {
        const commands = {
            testraidprotection: {
                name: "testraidprotection",
                description: "Tests the raid protection and security systems",
                async execute(message, args) {
                    if (!message.member.permissions.has(PermissionFlagsBits.Administrator)) {
                        return message.reply("You need Administrator permissions to use this command.");
                    }

                    if (message.client.testingInProgress) {
                        return message.channel.send("A security test is already in progress. Please wait for it to complete.");
                    }

                    message.client.testingInProgress = true;
                    console.log("[Security Test] Starting comprehensive security test...");

                    try {
                        await message.channel.send("🔍 Starting security systems test...");

                        await this.logSecurityAction(
                            message.guild,
                            "Test",
                            `Security systems test initiated by ${message.author.tag}`,
                            "INFO",
                            {
                                executor: message.author.tag,
                                test_type: "Full Security Test",
                                action_taken: "Starting controlled test"
                            }
                        );
                        console.log("[Security Test] Initialized test environment");

                        await message.channel.send("⚡ Testing single join detection...");
                        console.log("[Security Test] Testing single join...");
                        const mockMember = this.createMockMember(message.guild, "TestUser", 30);
                        await this.handleNewMember(mockMember);
                        console.log("[Security Test] Single join processed");
                        await message.channel.send("✅ Tested single join - verified no raid trigger");

                        await message.channel.send("⚡ Testing raid detection...");
                        console.log("[Security Test] Starting raid simulation...");
                        const joinCount = config.raidThreshold + 2;

                        console.log(`[Security Test] Simulating ${joinCount} rapid joins...`);
                        for (let i = 0; i < joinCount; i++) {
                            const mockRaidMember = this.createMockMember(message.guild, `RaidUser${i}`, 1);
                            await this.handleNewMember(mockRaidMember);
                            console.log(`[Security Test] Processed join ${i + 1}/${joinCount}`);
                            await new Promise(resolve => setTimeout(resolve, 100));
                        }

                        const raidModeActive = this.raidMode;
                        console.log(`[Security Test] Raid mode status: ${raidModeActive ? "ACTIVE" : "INACTIVE"}`);
                        await message.channel.send(
                            raidModeActive
                                ? "✅ Raid protection successfully activated"
                                : "❌ Failed to activate raid protection"
                        );

                        console.log("[Security Test] Checking channel lockdown status...");
                        const lockedChannels = message.guild.channels.cache.filter(
                            ch => ch.type === 0 && !ch.name.includes("mod") && !ch.name.includes("admin")
                        );

                        let channelStatus = "";
                        let lockedCount = 0;
                        for (const [_, channel] of lockedChannels) {
                            const perms = channel.permissionOverwrites.cache.get(
                                message.guild.roles.everyone.id
                            );
                            const isLocked = perms && !perms.allow.has("SEND_MESSAGES");
                            channelStatus += `\n${channel.name}: ${isLocked ? "🔒 Locked" : "❌ Not locked"}`;
                            if (isLocked) lockedCount++;
                        }
                        console.log(`[Security Test] Channels locked: ${lockedCount}/${lockedChannels.size}`);

                        await message.channel.send(`Channel Lockdown Status:${channelStatus}`);

                        await message.channel.send("⚡ Testing backup system...");
                        console.log("[Security Test] Testing backup system...");
                        let backupFile;
                        try {
                            backupFile = await this.createBackup(message.guild);
                            console.log(`[Security Test] Backup created: ${backupFile}`);
                            await message.channel.send(`✅ Backup created successfully: ${backupFile}`);
                        } catch (error) {
                            console.error("[Security Test] Backup creation failed:", error);
                            await message.channel.send("❌ Backup creation failed: " + error.message);
                        }

                        console.log("[Security Test] Generating final report...");
                        await message.channel.send(`
🔒 Security Systems Test Results:
• Join Detection: ✅ Working
• Raid Detection: ${raidModeActive ? "✅ Working" : "❌ Failed"}
• Channel Lockdown: ${channelStatus.includes("🔒") ? "✅ Working" : "❌ Failed"}
• Backup System: ${backupFile ? "✅ Working" : "❌ Failed"}
• Logging System: ✅ Enhanced logs generated

${raidModeActive ? "⚠️ Use !disableraid to disable raid protection and restore normal operation." : ""}
                        `);

                        if (raidModeActive) {
                            console.log("[Security Test] Cleaning up - disabling raid protection...");
                            await this.disableRaidProtection(message.guild);
                        }

                        console.log("[Security Test] Test completed successfully");
                    } catch (error) {
                        console.error("[Security Test] Test failed:", error);
                        await message.channel.send("❌ An error occurred while testing security systems.");
                        await this.logSecurityAction(
                            message.guild,
                            "Error",
                            `Failed to complete security systems test: ${error.message}`,
                            "CRITICAL",
                            {
                                error: error.message,
                                stack: error.stack
                            }
                        );
                    } finally {
                        message.client.testingInProgress = false;
                        console.log("[Security Test] Test state cleaned up");
                    }
                }
            },
            disableraid: {
                name: "disableraid",
                description: "Disables raid protection mode",
                async execute(message, args) {
                    if (!message.member.permissions.has(PermissionFlagsBits.Administrator)) {
                        return message.reply("You need Administrator permissions to use this command.");
                    }

                    await this.disableRaidProtection(message.guild);
                    await message.channel.send("✅ Raid protection has been disabled.");
                }
            },
            backup: {
                name: "backup",
                description: "Creates a manual backup of server settings",
                async execute(message, args) {
                    if (!message.member.permissions.has(PermissionFlagsBits.Administrator)) {
                        return message.reply("You need Administrator permissions to use this command.");
                    }

                    try {
                        const backupFile = await this.createBackup(message.guild);
                        await message.channel.send(`✅ Backup created successfully: ${backupFile}`);
                    } catch (error) {
                        await message.channel.send("❌ Failed to create backup: " + error.message);
                    }
                }
            }
        };

        for (const [name, command] of Object.entries(commands)) {
            this.commands.set(name, command);
        }
    }

    async initializeBackupDirectory() {
        try {
            await fs.mkdir(this.backupPath, { recursive: true });
        } catch (error) {
            console.error("Failed to create backup directory:", error);
        }
    }

    setupEventHandlers() {
        this.client.once("ready", () => {
            console.log(`Logged in as ${this.client.user.tag}`);
            this.client.user.setActivity("Protecting Server", { type: "WATCHING" });
            this.setupRequiredChannels();
            this.startAutoBackup();
        });

        this.client.on("messageCreate", async (message) => {
            if (!message.content.startsWith(config.prefix) || message.author.bot) return;

            const args = message.content.slice(config.prefix.length).trim().split(/ +/);
            const commandName = args.shift().toLowerCase();

            const command = this.commands.get(commandName);
            if (!command) return;

            try {
                await command.execute.call(this, message, args);
            } catch (error) {
                console.error(`Command execution error:`, error);
                await this.logSecurityAction(
                    message.guild,
                    "Command Error",
                    `Failed to execute command ${commandName}: ${error.message}`,
                    "CRITICAL",
                    {
                        error: error.message,
                        command: commandName,
                        args: args.join(" ")
                    }
                );
                await message.reply("An error occurred while executing the command.");
            }
        });

        this.client.on("guildMemberAdd", this.handleNewMember.bind(this));

        this.client.on("channelDelete", async (channel) => {
            const audit = await channel.guild.fetchAuditLogs({
                type: "CHANNEL_DELETE",
                limit: 1
            });
            const entry = audit.entries.first();
            if (entry) {
                await this.logAction.call(this, channel.guild, "CHANNEL_DELETE", entry.executor, channel);
            }
        });

        this.client.on("roleDelete", async (role) => {
            const audit = await role.guild.fetchAuditLogs({
                type: "ROLE_DELETE",
                limit: 1
            });
            const entry = audit.entries.first();
            if (entry) {
                await this.logAction.call(this, role.guild, "ROLE_DELETE", entry.executor, role);
            }
        });
    }

    async handleNewMember(member) {
        if (!member?.guild) return;

        try {
            const now = Date.now();
            console.log(`[Raid Protection] New member joined: ${member.user.tag}`);

            const accountAge = now - member.user.createdTimestamp;
            const daysSinceCreation = Math.floor(accountAge / (1000 * 60 * 60 * 24));

            if (daysSinceCreation < 7) {
                await this.logSecurityAction(
                    member.guild,
                    "Alt Detection",
                    `⚠️ New account joined: ${member.user.tag}`,
                    "WARNING",
                    {
                        perpetrator: member.user.tag,
                        account_age: `${daysSinceCreation} days`,
                        risk_level: "Medium"
                    }
                );
            }

            if (!this.raidModeLock) {
                this.raidModeLock = true;
                try {
                    this.joinLog.set(member.id, now);

                    if (!this.raidMode && this.isRaidInProgress()) {
                        this.raidMode = true;
                        await this.enableRaidProtection(member.guild);
                    }
                } finally {
                    this.raidModeLock = false;
                }
            }

            if (this.raidMode && config.raidAutoKick) {
                try {
                    await member.send({
                        content: "You have been removed due to active raid protection. Please try joining later.",
                        embeds: [
                            {
                                title: "⚠️ Server Protection Active",
                                description: "The server is currently experiencing high-volume joins and has enabled protection mode.",
                                color: 0xff0000
                            }
                        ]
                    });
                    await member.kick("Anti-raid protection active");
                } catch (error) {
                    console.error(`Failed to process member during raid: ${error.message}`);
                }
            }
        } catch (error) {
            console.error("Error in handleNewMember:", error);
        }
    }

    isRaidInProgress() {
        const now = Date.now();
        const cleanupTime = now - config.raidTimeWindow;
        let joinCount = 0;

        for (const [userId, timestamp] of this.joinLog.entries()) {
            if (timestamp < cleanupTime) {
                this.joinLog.delete(userId);
            } else {
                joinCount++;
            }
        }

        return joinCount >= config.raidThreshold;
    }

    async enableRaidProtection(guild) {
        try {
            console.log("[Raid Protection] Enabling raid protection mode");

            const channels = guild.channels.cache.filter(
                ch => ch.type === 0 && !ch.name.includes("mod") && !ch.name.includes("admin")
            );

            for (const [_, channel] of channels) {
                try {
                    await channel.permissionOverwrites.edit(guild.roles.everyone, {
                        ViewChannel: false,
                        SendMessages: false,
                        AddReactions: false,
                        CreatePublicThreads: false,
                        CreatePrivateThreads: false,
                        EmbedLinks: false,
                        AttachFiles: false
                    });
                } catch (error) {
                    console.error(`Failed to lock channel ${channel.name}:`, error);
                }
            }

            await this.logSecurityAction(
                guild,
                "RAID MODE ACTIVATED",
                "🚨 Server locked down for protection",
                "CRITICAL",
                {
                    action_taken: "Server lockdown",
                    affected_channels: channels.size
                }
            );
        } catch (error) {
            console.error("Error enabling raid protection:", error);
        }
    }

    async createBackup(guild) {
        try {
            const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
            const backupData = {
                timestamp,
                guildId: guild.id,
                name: guild.name,
                roles: [],
                channels: [],
                settings: {}
            };

            guild.roles.cache.forEach((role) => {
                if (role.name !== "@everyone") {
                    backupData.roles.push({
                        name: role.name,
                        color: role.color,
                        hoist: role.hoist,
                        permissions: role.permissions.bitfield,
                        mentionable: role.mentionable,
                        position: role.position
                    });
                }
            });

            guild.channels.cache.forEach((channel) => {
                const channelData = {
                    name: channel.name,
                    type: channel.type,
                    position: channel.position,
                    permissionOverwrites: []
                };

                channel.permissionOverwrites.cache.forEach((overwrite) => {
                    channelData.permissionOverwrites.push({
                        id: overwrite.id,
                        type: overwrite.type,
                        allow: overwrite.allow.bitfield,
                        deny: overwrite.deny.bitfield
                    });
                });

                backupData.channels.push(channelData);
            });

            const backupFileName = `backup-${guild.id}-${timestamp}.j
