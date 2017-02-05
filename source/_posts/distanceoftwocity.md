---
title: 根据经纬度计算两个城市间的距离
date: 2017-02-05 16:16:27
categories: golang
tags: [golang]
---

非常简单的golang小练习，这个问题最大的难点在于求两点的球面距离，所以这实际上是一道立体几何题，当然如果不记得公式咱们可以百度……废话不多说了，直接上代码。

city.go
``` go
package city

import (
    "encoding/csv"
    "math"
    "os"
    "strconv"
)

var (
    radius = float64(6371000)
    rad    = math.Pi / 180.0
)

// info 城市经纬度信息
type info struct {
    name string
    lng  float64 //经度
    lat  float64 //纬度
}

// Manager 城市管理
type Manager struct {
    citys map[string]*info
}

// NewCityConfig 读取坐标配置，创建管理器
func NewCityConfig(file string) (c *Manager, err error) {
    c = &Manager{}
    c.citys, err = c.parseCityInfo(file)
    if err != nil {
        return
    }

    return
}

// GetDistance 计算两个城市间的距离
func (c *Manager) GetDistance(city1 string, city2 string) (dist float64, err error) {
    c1 := c.citys[city1]
    c2 := c.citys[city2]
    if c1 == nil || c2 == nil {
        return 0.0, nil
    }

    dist = c.earthDistance(c1, c2)

    return
}

// earthDistance 计算距离
func (c *Manager) earthDistance(city1 *info, city2 *info) float64 {
    lng1 := city1.lng * rad
    lat1 := city1.lat * rad
    lng2 := city2.lng * rad
    lat2 := city2.lat * rad
    theta := lng2 - lng1
    dist := math.Acos(math.Sin(lat1)*math.Sin(lat2) + math.Cos(lat1)*math.Cos(lat2)*math.Cos(theta))
    return dist * radius
}

// parseCityInfo 载入城市经纬度
func (c *Manager) parseCityInfo(filepath string) (locations map[string]*info, err error) {
    f, err := os.Open(filepath)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    reader := csv.NewReader(f)
    record, err := reader.ReadAll()
    if err != nil {
        return nil, err
    }

    locations = make(map[string]*info, 0)
    for _, a := range record {
        city := &info{name: a[0]}
        lng, err := strconv.ParseFloat(a[1], 64)
        if err == nil {
            city.lng = lng
        }
        lat, err := strconv.ParseFloat(a[2], 64)
        if err == nil {
            city.lat = lat
        }
        locations[city.name] = city
    }

    return locations, nil
}

```

城市坐标使用csv格式存储：
city_info.csv
```
上海上海,121.48,31.22
上海嘉定,121.24,31.4
上海宝山,121.48,31.41
上海川沙,121.7,31.19
上海南汇,121.76,31.05
上海奉贤,121.46,30.92
上海松江,121.24,31
上海金山,121.16,30.89
上海青浦,121.1,31.15
上海崇明,121.4,31.73
……
```

city_test.go
``` go
package city

import (
    "fmt"
    "testing"
)

func TestGetDistance(t *testing.T) {
    c, err := NewCityConfig("city_info.csv")
    if err != nil {
        t.Errorf("new city config failed, %v", err)
    }

    var dist float64
    dist, err = c.GetDistance("广东深圳", "广东汕头")
    if err != nil {
        t.Errorf("GetDistance, %v", err)
    }
    fmt.Printf("distance of `广东深圳` and `广东汕头`: %.1f(m)\n", dist)
}

```

`go test`一下，得到如下结果：
```
distance of `广东深圳` and `广东汕头`: 281492.0(m)
    PASS
    ok      citydistant 0.410s
```
百度地图测距结果：
![](/images/20170205182525.jpg)

代码已放在github上：https://github.com/7byte/citydistant
