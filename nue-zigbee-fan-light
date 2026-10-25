import groovy.transform.Field
import groovy.json.JsonOutput
import java.util.concurrent.ConcurrentHashMap

/**
 * =======================================================================================
 * Nue Zigbee Light Fan Controller
 * Last modified: 2026-09-23
 * =======================================================================================
 */

@Field static final List supportedFanSpeeds = ["low", "medium", "high", "off"]
@Field static final List fanEndpoints = ["02", "03", "04"]
@Field static final List allEndpoints = ["01", "02", "03", "04"]
@Field static final Integer FAN_DEBOUNCE_MS = 250

// Per-device fan endpoint state, keyed by device.id -> [endpoint: Boolean]
@Field static ConcurrentHashMap<String, ConcurrentHashMap<String, Boolean>> fanStateCache = new ConcurrentHashMap<>()

metadata {
    definition (name: "Nue Zigbee Light Fan Controller", namespace: "mr_fy_hubitat", author: "mr_fy") {
        capability "Light"
        capability "FanControl"
        capability "Refresh"
        capability "Configuration"
        capability "Actuator"

        command "setSpeed", [[name: "Fan speed*", type: "ENUM", description: "Fan speed to set", constraints: supportedFanSpeeds]]

        attribute "fan", "STRING"
        attribute "switch", "STRING"

        fingerprint profileId:"C05E", endpointId:"01", inClusters:"0000,0004,0003,0006,0005,1000,0008", outClusters:"0000", model:"LXN-4S27LX1.0", manufacturer:"3A Smart Home DE", deviceJoinName: "Nue Fan Light Switch"
    }

    preferences {
        input name: "enableDebug", type: "bool", title: "Enable debug logging", defaultValue: false
        input name: "enableDesc", type: "bool", title: "Enable descriptionText logging", defaultValue: true
    }
}

/* ======== driver built-in callback methods ======== */

void installed() {
    log.info "installed..."
    device.updateSetting("enableDebug",[type:"bool", value: true])
    device.updateSetting("enableDesc",[type:"bool", value: true])
    initialize()
}

void uninstalled() {
    log.info "uninstalled..."
    fanStateCache.remove(device.id.toString())
}

void updated() {
    log.info "updated..."
    log.warn "debug logging is: ${enableDebug == true}"
    log.warn "description logging is: ${enableDesc == true}"
    state.remove("fanSpeeds")
    publishSupportedSpeeds()
    // Repopulate in-memory endpoint cache from the device
    sendHubCommand(new hubitat.device.HubMultiAction(refresh(), hubitat.device.Protocol.ZIGBEE))
}

void initialize() {
    log.info "initialize..."
    device.setName("NUE-FAN-LIGHT")
    updated()
    // configure() returns commands; they must be sent explicitly when not called by the platform
    sendHubCommand(new hubitat.device.HubMultiAction(configure(), hubitat.device.Protocol.ZIGBEE))
    unschedule()
    if (enableDebug) runIn(1800, "logsOff")
}

void parse(String description) {
    if (enableDebug) log.debug "parse description: ${description}"

    def now = new Date().format("yyyy MMM dd EEE h:mm:ss a", location.timeZone)
    sendEvent(name: "lastCheckin", value: now)

    Map descMap = zigbee.parseDescriptionAsMap(description)
    if (!descMap) return

    if (getClusterInt(descMap) != 0x0006) {
        if (enableDebug) log.debug "ignoring non-OnOff message: ${descMap}"
        return
    }

    String ep = normaliseEndpoint(descMap.endpoint ?: descMap.sourceEndpoint)
    String value = null

    if (description.startsWith("read attr")) {
        if (descMap.attrId == "0000") value = descMap.value
    } else if (description.startsWith("catchall")) {
        List data = descMap.data ?: []
        if (descMap.command == "0A" && data.size() >= 4 && data[0] == "00" && data[1] == "00") {
            // Report Attributes: attrId(2) type(1) value
            value = data[3]
        } else if (descMap.command == "01" && data.size() >= 5 && data[0] == "00" && data[1] == "00" && data[2] == "00") {
            // Read Attributes Response: attrId(2) status(1) type(1) value
            value = data[4]
        }
    }

    if (value == null) {
        if (enableDebug) log.debug "unhandled OnOff message on endpoint ${ep}: ${descMap}"
        return
    }

    boolean isOn = (value == "01")

    if (ep == "01") {
        handleLightReport(isOn)
    } else if (fanEndpoints.contains(ep)) {
        handleFanReport(ep, isOn)
    } else if (enableDebug) {
        log.debug "OnOff report from unexpected endpoint ${ep}"
    }
}

/* ======== report handlers ======== */

private void handleLightReport(boolean isOn) {
    String v = isOn ? "on" : "off"
    if (enableDesc && device.currentValue("switch") != v) log.info "${device.displayName} switch was set to ${v}"
    sendEvent(name: "switch", value: v)
    sendChildEvent(lightDni(), "switch", v)
}

private void handleFanReport(String ep, boolean isOn) {
    getFanCache().put(ep, isOn)
    // Debounce: interlocked endpoints report in a burst; publish once after it settles
    runInMillis(FAN_DEBOUNCE_MS, "publishFanSpeed")
}

// Scheduled handler - must not be private
void publishFanSpeed() {
    Map<String, Boolean> speeds = getFanCache()

    String newSpeed =
        speeds["02"] ? "high" :
        speeds["03"] ? "medium" :
        speeds["04"] ? "low" :
        "off"

    if (enableDebug) log.debug "publishFanSpeed() endpoints=${speeds} -> ${newSpeed}"
    if (enableDesc && device.currentValue("speed") != newSpeed) log.info "${device.displayName} fan was set to ${newSpeed}"
    sendEvent(name: "speed", value: newSpeed)
    sendEvent(name: "fan", value: newSpeed)
    sendChildEvent(fanDni(), "speed", newSpeed)
}

/* ======== capability commands ======== */

List<String> refresh() {
    if (enableDebug) log.debug "refresh()"
    List<String> cmds = []
    allEndpoints.each { ep ->
        cmds += zigbee.readAttribute(0x0006, 0x0000, [destEndpoint: Integer.parseInt(ep, 16)], 200)
    }
    return cmds
}

List<String> configure() {
    if (enableDebug) log.debug "configure()"
    createChildrenIfMissing()

    List<String> cmds = []
    allEndpoints.each { ep ->
        cmds << "zdo bind 0x${device.deviceNetworkId} 0x${ep} 0x01 0x0006 {${device.zigbeeId}} {}"
        cmds << "delay 300"
    }
    allEndpoints.each { ep ->
        cmds += zigbee.configureReporting(0x0006, 0x0000, 0x10, 1, 600, null, [destEndpoint: Integer.parseInt(ep, 16)])
        cmds << "delay 300"
    }
    cmds += refresh()
    return cmds
}

/* ======== custom commands and helpers ======== */

private Map<String, Boolean> getFanCache() {
    String id = device.id.toString()
    ConcurrentHashMap<String, Boolean> m = fanStateCache.get(id)
    if (m == null) {
        fanStateCache.putIfAbsent(id, new ConcurrentHashMap<String, Boolean>())
        m = fanStateCache.get(id)
    }
    return m
}

private void publishSupportedSpeeds() {
    String json = JsonOutput.toJson(supportedFanSpeeds)
    sendEvent(name: "supportedFanSpeeds", value: json)
    sendChildEvent(fanDni(), "supportedFanSpeeds", json)
}

private Integer getClusterInt(Map descMap) {
    if (descMap.clusterInt != null) return descMap.clusterInt as Integer
    String c = descMap.clusterId ?: descMap.cluster
    return c ? Integer.parseInt(c, 16) : null
}

private String normaliseEndpoint(def ep) {
    if (ep == null) return null
    String s = ep.toString().toUpperCase()
    return s.length() == 1 ? "0${s}" : s
}

private String lightDni() { "${device.deviceNetworkId}-Light" }
private String fanDni()   { "${device.deviceNetworkId}-Fan" }

private void createChildrenIfMissing() {
    if (!getChildDevice(lightDni())) addChildType("Light", "Light")
    if (!getChildDevice(fanDni()))   addChildType("Fan", "Fan")
}

def addChildType(String label, String type) {
    def devLabel = device.label ?: device.name
    if (type == "Light") {
        addChildDevice("hubitat","Generic Component Switch", "${device.deviceNetworkId}-${label}", [label: "${devLabel}-${label}", isComponent: true])
    } else {
        addChildDevice("hubitat","Generic Component Fan Control", "${device.deviceNetworkId}-${label}", [label: "${devLabel}-${label}", isComponent: true])
    }
}

def logsOff(){
    log.warn "debug logging disabled..."
    device.updateSetting("enableDebug",[value:"false",type:"bool"])
}

def sendChildEvent(n,a,v){
    def c = getChildDevice(n)
    if (c) c.sendEvent(name:a,value:v)
}

/* ======== Light and Fan control commands ======== */

def lightOn(){
    sendChildEvent(lightDni(),"switch","on")
    sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x01 0x0006 0x1 {}", hubitat.device.Protocol.ZIGBEE))
}

def lightOff(){
    sendChildEvent(lightDni(),"switch","off")
    sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x01 0x0006 0x0 {}", hubitat.device.Protocol.ZIGBEE))
}

def fanOff(){
    sendChildEvent(fanDni(),"speed","off")
    fanEndpoints.each{
        sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x${it} 0x0006 0x0 {}", hubitat.device.Protocol.ZIGBEE))
    }
}

def fanHighOn(){
    sendChildEvent(fanDni(),"speed","high")
    sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x02 0x0006 0x1 {}", hubitat.device.Protocol.ZIGBEE))
}

def fanMediumOn(){
    sendChildEvent(fanDni(),"speed","medium")
    sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x03 0x0006 0x1 {}", hubitat.device.Protocol.ZIGBEE))
}

def fanLowOn(){
    sendChildEvent(fanDni(),"speed","low")
    sendHubCommand(new hubitat.device.HubAction("he cmd 0x${device.deviceNetworkId} 0x04 0x0006 0x1 {}", hubitat.device.Protocol.ZIGBEE))
}

def setSpeed(s){
    switch(s){
        case "off":
            fanOff()
            break
        case "low":
            fanLowOn()
            break
        case "medium":
            fanMediumOn()
            break
        case "high":
            fanHighOn()
            break
    }
}

def componentSetSpeed(cd,s){
    if (enableDebug) log.debug "componentSetSpeed() called for ${cd.displayName} with speed ${s}"
    if(s == "on"){
        setSpeed("medium")
    } else {
        setSpeed(s)
    }
}

def cycleSpeed(){
    switch(device.currentValue("fan") ?: "off"){
        case "off": return fanLowOn()
        case "low": return fanMediumOn()
        case "medium": return fanHighOn()
        default: return fanOff()
    }
}

def componentCycleSpeed(cd){
    cycleSpeed()
}

void componentOn(cd){
    if (enableDebug) log.debug "componentOn() called for child device: ${cd.displayName}"
    if(cd.deviceNetworkId == lightDni()) {
        lightOn()
    } else {
        setSpeed("medium")
    }
}

void componentOff(cd){
    if(cd.deviceNetworkId == lightDni()) {
        lightOff()
    } else {
        fanOff()
    }
}

void on(){
    lightOn()
}

void off(){
    lightOff()
}

void componentRefresh(cd){
    if (enableDebug) log.debug "componentRefresh ${cd.displayName}"
    sendHubCommand(new hubitat.device.HubMultiAction(refresh(), hubitat.device.Protocol.ZIGBEE))
}
