## 5_13  

**地址**:`/share/org/PARATERA/MCC20258768/zrh/cesm1.2.1/cesm.model/models/atm/cam/src/dynamics/fv/cd_core.f90`

通过查询，可得到cd_code.F90中可以优化的地方：
1. **减少通信开销**  
在文件中，多个地方使用了`mp_spend4d_ns`和`mp_recv4d_ns`进行数据的发送和接收。这些通信操作可能会成为性能瓶颈，特别是在高并发情况下。  
可以对其进行以下优化：
    - **合并通信**：将多次通信合并为一次通信，减少通信次数
    - **非阻塞通信**：使用非阻塞的MPI通信(如MPI_lsend和MPI_lrevc)，以便在通信的同时进行计算
2. **优化边界通信**  
在文件中，边界数据的交换使用了`mp_send3d`和`mp_recv3d`，可以通过以下方式进行优化：
    - **减少边界数据量**：检查是否可以减少边界数据打大小或频率
    - **重叠通信与计算**：在边界通信的同时执行其他计算任务
3. **优化全局同步**  
在文件中，多次使用了`FVbarrierclock`进行全局同步。这些同步操作可能会导致性能下降，特别是在大规模并行时。可以考虑以下优化：
    - **减少同步次数**：检查是否可以减少不必要的全局同步
    - **分组同步**： 将全局同步改为局部分组同步，以减少同步开销。
4. **优化数据分解**  
文件中使用了二维和三维数据分解（如`yz`和`xy`分解）。可以见擦汗数据分析的策略是否最优，并根据计算和通信的附在重新分解方式

- 第一次优化尝试：
源代码：
```F90
if (ldiv2) then
  press = D0_5 * (grid%ak(k) + grid%ak(k+1) + (grid%bk(k) + grid%bk(k+1)) * D1E5)
  tau = D8_0 * (D1_0 + tanh(D1_0 * log(grid%ptop / press)))
  tau = max(D1_0, tau) / (D128_0 * abs(dt))

  ! Combine communication for cdx and cdy
  do j = js2g0, jn1g1
    fac = tau * ae / grid%cose(j)
    grid%cdx(j, k) = fac * grid%dp
    grid%cdy(j, k) = fac * grid%dl
  end do
end if

if (ldiv4) then
  tau4 = 0.01_r8 / (abs(dt))

  ! Combine communication for cdxdiv, cdydiv, cdx4, cdy4, and cdtau4
  do j = 1, jm
    grid%cdxdiv(j, k) = D1_0 / (grid%cose(j) * grid%dl)
    grid%cdydiv(j, k) = D1_0 / (grid%cose(j) * grid%dp)
  end do

  do j = js2g0, jn1g1
    fac = grid%dl * grid%cose(j)
    grid%cdx4(j, k) = D1_0 / (fac * fac)
    fac = grid%dp * grid%dp * grid%cose(j)
    grid%cdy4(j, k) = D1_0 / fac
    fac = grid%cose(j) * grid%dp * grid%dl
    grid%cdtau4(j, k) = -ae * tau4 * fac * fac
  end do
end if

```
使用合并通信，非阻塞通信修改后
```F90
if (ldiv2 .or. ldiv4) then
  ! Precompute press and tau for ldiv2
  if (ldiv2) then
    press = D0_5 * (grid%ak(k) + grid%ak(k+1) + (grid%bk(k) + grid%bk(k+1)) * D1E5)
    tau = D8_0 * (D1_0 + tanh(D1_0 * log(grid%ptop / press)))
    tau = max(D1_0, tau) / (D128_0 * abs(dt))
  end if

  ! Use a single loop to compute all coefficients
  do j = 1, jm
    if (ldiv2) then
      fac = tau * ae / grid%cose(j)
      grid%cdx(j, k) = fac * grid%dp
      grid%cdy(j, k) = fac * grid%dl
    end if

    if (ldiv4) then
      grid%cdxdiv(j, k) = D1_0 / (grid%cose(j) * grid%dl)
      grid%cdydiv(j, k) = D1_0 / (grid%cose(j) * grid%dp)
    end if
  end do

  ! Compute additional coefficients for ldiv4
  if (ldiv4) then
    do j = js2g0, jn1g1
      fac = grid%dl * grid%cose(j)
      grid%cdx4(j, k) = D1_0 / (fac * fac)
      fac = grid%dp * grid%dp * grid%cose(j)
      grid%cdy4(j, k) = D1_0 / fac
      fac = grid%cose(j) * grid%dp * grid%dl
      grid%cdtau4(j, k) = -ae * tau4 * fac * fac
    end do
  end if
end if
```

结果：报错好多