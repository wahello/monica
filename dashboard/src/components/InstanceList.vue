<template>
    <div>
        <List border>
            <ListItem v-for="item in instances" :key="item.Name">
                <ListItemMeta :title="item.Name" :description="item.Path"></ListItemMeta>
                <template v-if="typeof item.Info == 'string'">{{item.Info}}</template>
                <template v-else-if="item.Info!=null">
                    引擎版本：{{item.Info.Version}} 启动时间：
                    <StartTime :value="item.Info.StartTime"></StartTime>
                </template>
                <template slot="action">
                    <li @click="changeConfig(item)">
                        <Icon type="ios-settings"/>
                        修改配置
                    </li>
                    <li v-if="hasGateway(item)" @click="openGateway(item)">
                        <Icon type="md-browsers"/>
                        管理界面
                    </li>
                    <li @click="currentItem=item,showRestart=true">
                        <Icon type="ios-refresh"/>
                        重启
                    </li>
                    <li @click="shutdown(item)">
                        <Icon type="ios-power"/>
                        关闭
                    </li>
                </template>
            </ListItem>
        </List>
        <Modal v-model="showRestart" title="重启选项" @on-ok="restart">
            <Checkbox v-model="update">go get -u</Checkbox>
            <Checkbox v-model="build">go build</Checkbox>
        </Modal>
        <Modal v-model="showConfig" title="修改实例配置" @on-ok="submitConfigChange">
            <i-input type="textarea" v-model="currentConfig" :rows="20"></i-input>
        </Modal>
    </div>
</template>

<script>
    import toml from "@iarna/toml"
    import StartTime from "./StartTime"

    export default {
        name: "InstanceList",
        components: {StartTime},
        data() {
            return {
                instances: [],
                showRestart: false,
                update: false,
                build: false,
                showConfig: false,
                currentItem: null,
                currentConfig: ""
            }
        },
        mounted() {
            window.ajax.getJSON("/api/list").then(x => {
                for (let name in x) {
                    let instance = x[name]
                    instance.Config = toml.parse(instance.Config)
                    if (this.hasGateway(instance)) {
                        window.ajax.getJSON(this.gateWayHref(instance) + "/api/sysInfo").then(x => {
                            instance.Info = x
                        }).catch(() => {
                            instance.Info = "无法访问实例"
                        })
                    } else {
                        instance.Info = "实例未配置网关插件"
                    }
                    this.instances.push(instance)
                }
                // this.instances = x;
            });
        },
        methods: {
            changeConfig(item) {
                this.showConfig = true
                this.currentItem = item
                this.currentConfig = toml.stringify(item.Config)
            },
            submitConfigChange() {
                try {
                    this.currentItem.Config = toml.parse(this.currentConfig)
                    window.ajax.post("/api/updateConfig?instance=" + this.currentItem.Name, this.currentConfig).then(x => {
                        if (x == "success") {
                            this.$Message.success("更新成功！")
                        } else {
                            this.$Message.error(x)
                        }
                    }).catch(e => {
                        this.$Message.error(e)
                    })
                } catch (e) {
                    this.$Message.error(e)
                }
            },
            openGateway(item) {
                window.open(this.gateWayHref(item), '_blank')
            },
            hasGateway(item) {
                return item.Config.Plugins.hasOwnProperty("GateWay")
            },
            gateWayHref(item) {
                return "http://" + location.hostname + ":" + item.Config.Plugins.GateWay.ListenAddr.split(":").pop()
            },
            restart() {
                let item = this.currentItem
                const msg = this.$Message.loading({
                    content: 'restart ' + item.Name + '...',
                    duration: 0
                });
                let arg = item.Name
                if (this.update) {
                    arg += "&update=true"
                }
                if (this.build) {
                    arg += "&build=true"
                }
                const es = new EventSource("/api/restart?instance=" + arg)
                es.onmessage = evt => {
                    if (evt.data == "success") {
                        this.$Message.success("重启成功！")
                        es.onerror = null
                        es.close()
                        msg()
                    } else {
                        this.$Message.info(evt.data)
                    }
                }
                es.addEventListener("failed", evt => {
                    this.$Message.error(evt.data)
                    msg()
                })
                es.onerror = e => {
                    this.$Message.error(e.toString());
                    msg()
                    es.close()
                }
            },
            shutdown(item) {
                window.ajax.get("/api/shutdown?instance=" + item.Name).then(x => {
                    if (x == "success") {
                        this.$Message.success("已关闭实例")
                    } else {
                        this.$Message.error(x)
                    }
                })
            },
        }
    }
</script>

<style scoped>

</style>