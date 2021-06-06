# Fake GPU Device Plugin

> To test GPU without any phisical GPUs

- [Fake GPU Device Plugin](#fake-gpu-device-plugin)
  - [config.json](#configjson)
  - [main.go](#maingo)
  - [nvidia.go](#nvidiago)
  - [server.go](#servergo)
  - [device.go](#devicego)
  - [nvml.go](#nvmlgo)

## config.json

```json
{
    "id": "a",
    "count": 3
}
```

- `id` : Assign unique character per node
- `count` : Allocatable GPU count

## main.go

```diff
+	"github.com/NVIDIA/gpu-monitoring-tools/bindings/go/nvml"

func start(c *cli.Context) error {
	log.Println("Loading NVML")
+	// if err := nvml.Init(); err != nil {
+	// 	log.SetOutput(os.Stderr)
+	// 	log.Printf("Failed to initialize NVML: %v.", err)
+	// 	log.Printf("If this is a GPU node, did you set the docker default runtime to `nvidia`?")
+	// 	log.Printf("You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites")
+	// 	log.Printf("You can learn how to set the runtime at: https://github.com/NVIDIA/k8s-device-plugin#quick-start")
+	// 	log.Printf("If this is not a GPU node, you should set up a toleration or nodeSelector to only deploy this plugin on GPU nodes")
+	// 	if failOnInitErrorFlag {
+	// 		return fmt.Errorf("failed to initialize NVML: %v", err)
+	// 	}
+	// 	select {}
+	// }
+	// defer func() { log.Println("Shutdown of NVML returned:", nvml.Shutdown()) }()
```

## nvidia.go

```diff
import (
	"fmt"
	"log"
	"os"
	"strings"
+	"time"


func checkHealth(stop <-chan interface{}, devices []*Device, unhealthy chan<- *Device) {
	disableHealthChecks := strings.ToLower(os.Getenv(envDisableHealthChecks))
	if disableHealthChecks == "all" {
		disableHealthChecks = allHealthChecks
	}
	if strings.Contains(disableHealthChecks, "xids") {
		return
	}

+	// eventSet := nvml.NewEventSet()
+	// defer nvml.DeleteEventSet(eventSet)
+
+	// for _, d := range devices {
+	// 	gpu, _, _, err := nvml.ParseMigDeviceUUID(d.ID)
+	// 	if err != nil {
+	// 		gpu = d.ID
+	// 	}
+
+	// 	err = nvml.RegisterEventForDevice(eventSet, nvml.XidCriticalError, gpu)
+	// 	if err != nil && strings.HasSuffix(err.Error(), "Not Supported") {
+	// 		log.Printf("Warning: %s is too old to support healthchecking: %s. Marking it unhealthy.", d.ID, err)
+	// 		unhealthy <- d
+	// 		continue
+	// 	}
+	// 	check(err)
+	// }

	for {
		select {
		case <-stop:
			return
		default:
		}

+		time.Sleep(500 * time.Microsecond)

+		// e, err := nvml.WaitForEvent(eventSet, 5000)
+		// if err != nil && e.Etype != nvml.XidCriticalError {
+		// 	continue
+		// }
+
+		// // FIXME: formalize the full list and document it.
+		// // http://docs.nvidia.com/deploy/xid-errors/index.html#topic_4
+		// // Application errors: the GPU should still be healthy
+		// if e.Edata == 31 || e.Edata == 43 || e.Edata == 45 {
+		// 	continue
+		// }
+
+		// if e.UUID == nil || len(*e.UUID) == 0 {
+		// 	// All devices are unhealthy
+		// 	log.Printf("XidCriticalError: Xid=%d, All devices will go unhealthy.", e.Edata)
+		// 	for _, d := range devices {
+		// 		unhealthy <- d
+		// 	}
+		// 	continue
+		// }
+
+		// for _, d := range devices {
+		// 	// Please see https://github.com/NVIDIA/gpu-monitoring-tools/blob/148415f505c96052cb3b7fdf443b34ac853139ec/bindings/go/nvml/nvml.h#L1424
+		// 	// for the rationale why gi and ci can be set as such when the UUID is a full GPU UUID and not a MIG device UUID.
+		// 	gpu, gi, ci, err := nvml.ParseMigDeviceUUID(d.ID)
+		// 	if err != nil {
+		// 		gpu = d.ID
+		// 		gi = 0xFFFFFFFF
+		// 		ci = 0xFFFFFFFF
+		// 	}
+
+		// 	if gpu == *e.UUID && gi == *e.GpuInstanceId && ci == *e.ComputeInstanceId {
+		// 		log.Printf("XidCriticalError: Xid=%d on Device=%s, the device will go unhealthy.", e.Edata, d.ID)
+		// 		unhealthy <- d
+		// 	}
+		// }
	}
}
```

## server.go

```diff
+// sskk: K8s에서 GPU 할당을 요청하면 이 함수가 호출된다.
// GetPreferredAllocation returns the preferred allocation from the set of devices specified in the request
func (m *NvidiaDevicePlugin) GetPreferredAllocation(ctx context.Context, r *pluginapi.PreferredAllocationRequest) (*pluginapi.PreferredAllocationResponse, error) {
	response := &pluginapi.PreferredAllocationResponse{}
	for _, req := range r.ContainerRequests {
		available, err := gpuallocator.NewDevicesFrom(req.AvailableDeviceIDs)
		if err != nil {
			return nil, fmt.Errorf("Unable to retrieve list of available devices: %v", err)
		}

		required, err := gpuallocator.NewDevicesFrom(req.MustIncludeDeviceIDs)
		if err != nil {
			return nil, fmt.Errorf("Unable to retrieve list of required devices: %v", err)
		}

		allocated := m.allocatePolicy.Allocate(available, required, int(req.AllocationSize))

		var deviceIds []string
		for _, device := range allocated {
			deviceIds = append(deviceIds, device.UUID)
		}

		resp := &pluginapi.ContainerPreferredAllocationResponse{
			DeviceIDs: deviceIds,
		}

		response.ContainerResponses = append(response.ContainerResponses, resp)
	}
	return response, nil
}
```

## device.go

`vendor/github.com/NVIDIA/go-gpuallocator/gpuallocator`

```diff
// Create a list of Devices from all available nvml.Devices.
func NewDevices() ([]*Device, error) {
	count, err := nvml.GetDeviceCount()
	if err != nil {
		return nil, fmt.Errorf("error calling nvml.GetDeviceCount: %v", err)
	}

	devices := []*Device{}
	for i := 0; i < int(count); i++ {
-        device, err := nvml.NewDevice(uint(i))
+		device, err := nvml.NewDeviceLite(uint(i))
		if err != nil {
			return nil, fmt.Errorf("error creating nvml.Device %v: %v", i, err)
		}

		devices = append(devices, &Device{device, i, make(map[int][]P2PLink)})
	}

+	// for i, d1 := range devices {
+	// 	for j, d2 := range devices {
+	// 		if d1 != d2 {
+	// 			p2plink, err := nvml.GetP2PLink(d1.Device, d2.Device)
+	// 			if err != nil {
+	// 				return nil, fmt.Errorf("error getting P2PLink for devices (%v, %v): %v", i, j, err)
+	// 			}
+	// 			if p2plink != nvml.P2PLinkUnknown {
+	// 				d1.Links[d2.Index] = append(d1.Links[d2.Index], P2PLink{d2, p2plink})
+	// 			}
+
+	// 			nvlink, err := nvml.GetNVLink(d1.Device, d2.Device)
+	// 			if err != nil {
+	// 				return nil, fmt.Errorf("error getting NVLink for devices (%v, %v): %v", i, j, err)
+	// 			}
+	// 			if nvlink != nvml.P2PLinkUnknown {
+	// 				d1.Links[d2.Index] = append(d1.Links[d2.Index], P2PLink{d2, nvlink})
+	// 			}
+	// 		}
+	// 	}
+	// }

	return devices, nil
}
```

## nvml.go

`vendor/github.com/NVIDIA/gpu-monitoring-tools/bindings/go/nvml`

```diff
import (
	"bytes"
+	"encoding/json"
	"errors"
	"fmt"
	"io/ioutil"
	"runtime"
	"strconv"
	"strings"
)

+func getFakeDeviceCount() uint {
+	data, err := ioutil.ReadFile("config.json")
+	if err != nil {
+		print(err)
+		return 0
+	}
+
+	var config interface{}
+	err = json.Unmarshal(data, &config)
+	if err != nil {
+		print(err)
+		return 0
+	}
+
+	m := config.(map[string]interface{})
+	return uint(m["count"].(float64))
+}

func GetDeviceCount() (uint, error) {
+	return getFakeDeviceCount(), nil
+	// return deviceGetCount()
}

+func deviceGetUUID(idx uint) *string {
+	var uuid [0x60]C.char
+
+	data, err := ioutil.ReadFile("config.json")
+	if err != nil {
+		print(err)
+		return nil
+	}
+
+	var config interface{}
+	err = json.Unmarshal(data, &config)
+	if err != nil {
+		print(err)
+		return nil
+	}
+
+	m := config.(map[string]interface{})
+	uuid[0] = C.char(m["id"].(string)[0])
+
+	for i := 1; i < 0x60; i++ {
+		// uuid[i] = C.char(i)
+		uuid[i] = C.char('0' + idx)
+	}
+
+	return stringPtr(&uuid[0])
+}

+func deviceGetPciInfo(idx uint) *string {
+	//var pci C.nvmlPciInfo_t
+
+	// r := C.nvmlDeviceGetPciInfo(h.dev, &pci)
+	// if r == C.NVML_ERROR_NOT_SUPPORTED {
+	// 	return nil, nil
+	// }
+	var i C.char
+	i = C.char(idx)
+	return stringPtr(&i)
+}

func NewDeviceLite(idx uint) (device *Device, err error) {
-	defer func() {
-		if r := recover(); r != nil {
-			err = r.(error)
-		}
+	// defer func() {
+	// 	if r := recover(); r != nil {
+	// 		err = r.(error)
+	// 	}
+	// }()
+	var h handle
+	//h, err := deviceGetHandleByIndex(idx)
+	// assert(err)
+
+	// device, err = newDeviceLite(h)
+	// assert(err)
+
+	// return device, err
+	// uuid, err := h.deviceGetUUID()
+	uuid := deviceGetUUID(idx)
+
+	// assert(err)
+	var minor *uint
+	var m C.uint
+	m = 0
+	minor = uintPtr(m)
+
+	// minor, err := h.deviceGetMinorNumber()
+	// assert(err)
+	// busid, err := h.deviceGetPciInfo()
+	// assert(err)
+	busid := deviceGetPciInfo(idx)
+
+	// if minor == nil || busid == nil || uuid == nil {
+	// 	return nil, ErrUnsupportedGPU
+	// }
+
+	path := fmt.Sprintf("/dev/nvidia%d", *minor)
+	// node, err := numaNode(*busid)
+	// assert(err)
+	var node *uint
+
+	return &Device{
+		handle:      h,
+		UUID:        *uuid,
+		Path:        path,
+		CPUAffinity: node,
+		PCI: PCIInfo{
+			BusID: *busid,
+		},
+	}, nil
}

+// func NewDeviceLite(idx uint) (device *Device, err error) {
+// 	defer func() {
+// 		if r := recover(); r != nil {
+// 			err = r.(error)
+// 		}
+// 	}()
+
+// 	h, err := deviceGetHandleByIndex(idx)
+// 	assert(err)
+
+// 	device, err = newDeviceLite(h)
+// 	assert(err)
+
+// 	return device, err
+// }
```